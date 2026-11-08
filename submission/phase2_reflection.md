# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->393.205<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.962<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->416782.0<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13440.7<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4256<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2830.249 ms, 2 threads: 2007.56 ms, 4 threads: 2118.998 ms, 8 threads: 2135.875 ms. The fastest was 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version is slow because of Python, not because of the math. Every multiplication is done one by one by the interpreter and each one has a lot of overhead. The vectorized version sends the whole matrices to NumPy, which uses BLAS. BLAS is compiled code that uses the SIMD (AVX) instructions of the CPU, so it computes several floats at once. That's why it went from 393.205 ms to 0.962 ms.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
2 threads was the fastest. With 1 thread a step took 2830.249 ms and with 2 threads 2007.56 ms, so it's faster but not twice as fast. With 4 and 8 threads it got a bit slower again (2118.998 ms and 2135.875 ms). Colab only has 2 vCPUs, so when there are more threads than that they have to share the same cores, and they lose time switching and waiting for each other at the end of each operation. Also some parts of the step can't be parallelized, and both threads use the same memory.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Most of a transformer training step is matrix multiplications, and a lot of them can be done at the same time because the positions in the batch are independent. A GPU like the T4 has thousands of cores for this. The CPU took 416782.0 ms for 100 steps and the GPU 13440.7 ms, so about 31 times faster. Generating text one token at a time with a batch of 1 would not really benefit, because the work is too small and the time spent launching kernels and copying data to the GPU takes over.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
It depends on the experiment. With vectorization I didn't really hit a limit, the gain (393.205 ms to 0.962 ms) mostly comes from not going through Python anymore. With threads I hit several limits. Some of the work can't be split, so 2 threads only gave 1.41x (2830.249 ms to 2007.56 ms), which is Amdahl's law. Then there's the hardware: Colab has only 2 vCPUs, so 4 or 8 threads have nothing more to run on. And the threads have to synchronize, which costs more than it gains when there are too many of them (2135.875 ms at 8 threads vs 2007.56 ms at 2). For the GPU the problem is moving data. 31x is good but a T4 can do more. Our model is small (4.9M parameters) and each step copies a batch from the CPU to the GPU and launches a lot of small kernels, so the GPU isn't fully used (13440.7 ms for 100 steps).
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would run the CPU and GPU with different batch sizes, like 8, 32, 128 and 256. If the GPU speedup gets a lot bigger than 31x with big batches, it means the GPU was slowed down by overhead before. If it stays around 31x, I was wrong about that.
<!--/answer-->
