# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->268.0<!--/answer--> | <!--answer:vm_warm_median_ms-->210.5<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2192.5<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->634.9<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->2020.8<!--/answer--> | <!--answer:function_warm_median_ms-->638.1<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->

[OK  ] vm        cold=  268.0ms  warm_median=  210.5ms
            #############
[OK  ] cloudrun  cold= 2192.5ms  warm_median=  634.9ms
            ########################################
[OK  ] function  cold= 2020.8ms  warm_median=  638.1ms
            ########################################

<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The IaaS VM creates nothing automatically under load; its capacity is strictly limited by the e2-micro machine size. 
Cloud Run dynamically spins up additional container instances to handle increasing incoming traffic, up to the limit set by the user with --max-instances=3 flag. 
Similarly, Cloud Functions automatically provision new function instance runtimes to absorb request spikes, also bounded strictly by the configured max instances setting.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
For the IaaS VM, deployment effort was the highest: I did manual VPC and subnet creation, firewall rules, VM instance provisioning, SSH configuration, installing Docker and eventually running the container manually. 
Cloud Run required medium effort: I wrote a Dockerfile, built a container image, pushed it to Artifact Registry and deployed it as a managed service. 
For Cloud Functions instead, the effort was minimal: I deployed Python source code directly via gcloud functions deploy without maintaining container images, Dockerfiles, network interfaces or web servers.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-1068278423417.us-central1.run.app
LAST DEPLOYED BY: Alberto.Actis-Grosso@eurecom.fr
LAST DEPLOYED AT: 2026-10-25T14:54:57.309917Z

✔
SERVICE: echo-fn
REGION: us-central1
URL: https://echo-fn-1068278423417.us-central1.run.app
LAST DEPLOYED BY: service-1068278423417@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-25T15:06:16.349138Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output of gcloud run services list contains both echo-bot and echo-fn as active
Cloud Run services. 
This clearly shows that Gen 2 Cloud Functions are actually just Cloud Run services under the hood! When you deploy a Gen 2 function, GCP automatically packages the Python code into a container and runs it on Cloud Run. Because both platforms share the exact same underlying container execution environment and scale-to-zero lifecycle mechanics, their cold-start latencies (~2.0-2.2s) and warm median response times (~635-638ms) are almost identical. 
The main difference is just the abstraction level: with Cloud Run you build and manage the Docker image yourself, while Cloud Functions handles the container build process for you.
<!--/answer-->
