# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->29,5<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0,03<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4,3<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->15<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranking: local (0.03 s) < Spark (4.3 s) < cloud MapReduce (29.5 s). The like-for-like comparison is local vs cloud: same word count, same 2221 words. Local is pure in-memory computation, Spark mostly start-up. The cloud job pays overhead: 16 HTTP calls with cold starts, Cloud Storage reads and writes for splits and shuffle files, workflow scheduling and retry backoff. Spark does more work two reduceByKey and a join on one machine, so the cloud time is coordination and I/O, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
4 stages means 3 shuffles: the reduceByKey in term_freq, the reduceByKey in doc_freq, and the join in build_tfidf_rdd. I counted 15 retries for 16 tasks (31 attempts, ~48% failed). That is more than the 30% -chaos 0.3 should give (about 7 retries), but with so few tasks the variance is high.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
 Transformations (lazy): parallelize, flatMap, map, reduceByKey and join. The only action is collect. Calling build_tfidf_rdd returned instantly because it only recorded the lineage graph shown by `toDebugString; nothing was computed. collect triggered the whole job: Spark split the lineage into 4 stages and 12 tasks, ran them in 4.3 s and returned 5975 rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first reduceByKey produces ShuffledRDD[3]. If one of its partitions were lost, Spark would recompute only that partition from its lineage: it re-reads the stage-0 shuffle files, or re-runs the stage-0 tasks parallelize + flatMap if those are gone. Everything else is reused. Recovery starts at the first shuffle boundary, after stage 0.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows: the mr-wordcount workflow defined in workflow.yaml<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The mr-worker Cloud Run function (2nd gen), code in main.py calling mapreduce.py<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage: the <project>-eurecomgpt bucket (input/, shuffle/, output/)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition() in mapreduce.py: zlib.crc32(word) % num_reducers<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the end of the map_phase and parallel  step: it only completes when every branch has finished, so reduce_phase cannot start before the last map task, including retries, returns.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each map task writes to the same fixed paths  and overwrites them, and the simulated crash happens before any work. So re-running a task gives the same files, and retrying is safe. If run_map appended instead, a retried task would add its pairs a second time, so reducers would double-count words and the result would no longer match the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
in logs exlorer , filtering on 'mr-worker' with 'httpRequest.statu=503' , i found 15 POST request answered 503, all around 23:43 , each lasting only 2-100 ms ,the crash append befor work. in the execution view , the failed call_map/call_reduce steps were retried with exponential backoff. the job still finished SUCCEEDED in 29,5vs 23 without KAO and the output was identical to the local reference'identical:true,2221 worlds 
<!--/answer-->
