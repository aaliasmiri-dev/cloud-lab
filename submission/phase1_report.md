# Phase 1 — Comparison Report

# Phase 1 - Comparison report

## 1. Setup facts

<!--answer:image_size_mb-->
50
<!--/answer-->

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->251.7<!--/answer--> | <!--answer:vm_warm_median_ms-->211.9<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2608.9<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->639.8<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1854.4<!--/answer--> | <!--answer:function_warm_median_ms-->641.8<!--/answer--> |

<!--answer:latency_plot-->
```
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  251.7ms  warm_median=  211.9ms
            #############
[OK  ] cloudrun  cold= 2608.9ms  warm_median=  639.8ms
            ########################################
[OK  ] function  cold= 1854.4ms  warm_median=  641.8ms
            ########################################
------------------------------------------------------------
```
I waited about 15 minutes without touching any endpoint before running measure.py, so Cloud Run and the function were scaled to zero. That's why their first request is so slow (2.6 s and 1.9 s): a new container had to be started. The VM doesn't really have a cold start since the container is always running, the first request is just a bit slower (251.7 vs 211.9 ms).

Something I didn't expect: looking at the raw values in phase1_report.json, the warm latencies of Cloud Run and the function are not regular at all. Most requests are around 640 ms but a few are around 120 ms. I think this comes from the HTTPS layer in front of the services (new connection / TLS or different Google front ends), not from the app itself, because the VM (plain HTTP on its IP) stays between 210 and 216 ms.
<!--/answer-->

---

## 3. Per-platform analysis

<!--answer:scaling_behaviour-->
VM: nothing is created automatically. It's one e2-micro with one container running gunicorn with 1 worker and 8 threads, so after 8 requests at the same time the others have to wait, and with 100 requests it would probably slow down a lot or time out. The limit is just the size of the machine I chose. To scale I would have to set up an instance group and a load balancer myself.

Cloud Run: new instances of my container are started automatically when there is more traffic than one instance can handle (80 concurrent requests per instance by default). The hard limit is 3 instances because I deployed with --max-instances=3.

Function: same idea as Cloud Run, new instances are created, but each instance only handles 1 request at a time, so 10 requests at once means around 10 instances. I didn't set a max, so the limit is the default one.
<!--/answer-->

<!--answer:deployment_effort-->
The VM was clearly the most work. I had to create the VPC, the subnet and two firewall rules (8080 for the app and 22 for SSH, both applied with the echo-http tag), give the VM's service account the Artifact Registry reader role, create the VM (I used a pd-standard disk to stay in the free tier), then SSH in to install Docker, configure authentication and run the container. And after that I would still be responsible for OS updates and keeping it running.

For Cloud Run, the main work was the Dockerfile (my first build failed because I forgot RUN before pip install) and pushing the image with Cloud Build. I also got a 403 because the default compute service account didn't have the rights to read the source, I had to add the cloudbuild.builds.builder role. After that, the deploy itself was a single command.

The function was the easiest: no Dockerfile, I just pointed gcloud to the source folder and gave the entry point.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

<!--answer:run_services_list-->
```
SERVICE: echo
REGION: us-central1
URL: https://echo-246373107848.us-central1.run.app
LAST DEPLOYED BY: service-246373107848@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-26T23:57:36.663669Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-246373107848.us-central1.run.app
LAST DEPLOYED BY: smiriaalia@gmail.com
LAST DEPLOYED AT: 2026-10-26T23:56:27.072741Z
```
<!--/answer-->

<!--answer:convergence_explanation-->
I only deployed one Cloud Run service myself (echo-bot), but the list shows two. The second one, echo, is my Cloud Function, and it was deployed by gcf-admin-robot, which is the Cloud Functions service account and not me. So a gen 2 function actually runs on Cloud Run. When I deployed it, Google built a container image from my source code (I saw it went into a gcf-artifacts repository in the deploy output) and then deployed that image on Cloud Run.

This explains why the two have almost the same numbers: same platform, same scale to zero, same way of starting an instance (warm medians 639.8 vs 641.8 ms).

The real difference is what I control. With Cloud Run I made the image myself (Dockerfile, base image, gunicorn with 8 threads). With the function I only gave the code and Google generated the rest, including a concurrency of 1 request per instance (maxInstanceRequestConcurrency: 1 in the deploy output) instead of 80 for Cloud Run.
<!--/answer-->