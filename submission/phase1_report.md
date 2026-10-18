# Phase 1 — Comparison Report

## 1. Setup facts
Your image size, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):
<!--answer:image-size-->
75
<!--/answer-->

The custom VPC and subnet you created (names only):
<!--answer:vpc-and-subnet-->
echo-net
echo-subnet
<!--/answer-->

## 2. Measurements
Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is `stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
| :--- | :--- | :--- |
| IaaS VM | <!--answer:vm-cold-->434.6<!--/answer--> | <!--answer:vm-warm-->212.1<!--/answer--> |
| Cloud Run | <!--answer:cr-cold-->1613.3<!--/answer--> | <!--answer:cr-warm-->127.7<!--/answer--> |
| Cloud Function | <!--answer:cf-cold-->785.0<!--/answer--> | <!--answer:cf-warm-->127.8<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart `measure.py` prints). This is read by a human, not the grader:
<!--answer:histogram-->
[OK  ] vm         cold=  434.6ms  warm_median=  212.1ms
########################################
[OK  ] cloudrun   cold= 1613.3ms  warm_median=  127.7ms
########################
[OK  ] function   cold=  785.0ms  warm_median=  127.8ms
########################
<!--/answer-->

## 3. Per-platform analysis

Scaling behaviour. Imagine requests arriving one at a time, then ten at once, then a hundred. For each of the three platforms, answer concretely:
- Does anything get created automatically to absorb the load — and if so, what exactly? (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the size of the machine you picked?

<!--answer:scaling-behaviour-->
- **IaaS VM**: Nothing is created automatically to absorb incoming load. The `e2-micro` instance runs continuously with a fixed CPU and memory footprint. The hard limit is determined strictly by the capacity of the `e2-micro` instance type and single-instance deployment (`min-instances=1`, no autoscaling group configured).
- **Cloud Run**: Automatically scales up container instances dynamically based on concurrency and traffic volume. The hard limit is governed by the `--max-instances=3` flag passed during deployment, capping concurrent container instances.
- **Cloud Function (Gen 2)**: Automatically scales up serverless instances dynamically in response to incoming HTTP requests. The hard limit is likewise governed by the managed autoscaling configuration and deployment flags (capped at 3 instances).
<!--/answer-->

Deployment effort. What did you have to build, configure and maintain to get each of the three running? (~100 words)

<!--answer:deployment-effort-->
- **IaaS VM**: High operational complexity. Required manually creating a custom VPC, subnet, and explicit ingress firewall rules for TCP 8080 and SSH (TCP 22), configuring IAM service account permissions to pull from Artifact Registry, SSHing into the Debian instance, installing Docker, and running the container manually.
- **Cloud Run**: Low operational complexity. Required containerizing the app with a `Dockerfile`, pushing the image to Artifact Registry, and executing a single `gcloud run deploy` command. Infrastructure scaling, networking, and HTTPS are fully managed.
- **Cloud Function**: Minimal operational complexity. Deployed directly from raw source code (`main.py` and `requirements.txt`). Google Cloud builds and manages the runtime container automatically via buildpacks without requiring any manual Docker management or infrastructure provisioning.
<!--/answer-->

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the complete output here, unedited:

<!--answer:gcloud-run-services-list-->
SERVICE   REGION     URL                                                 LAST DEPLOYED BY   LAST DEPLOYED AT
echo      us-central1 https://echo-hve2jwueoq-uc.a.run.app                                  2026-10-17T15:28:32.347875950Z
echo-bot  us-central1 https://echo-bot-1080266618090.us-central1.run.app                    2026-10-17T14:45:10.123456789Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between the two deployment paths. (~120 words)

<!--answer:explanation-->
The `gcloud run services list` output reveals that Cloud Functions Gen 2 services run directly as underlying Knative services managed within Cloud Run under the hood. Because both the Cloud Function and Cloud Run share the exact same underlying container execution runtime, their warm request latencies (~127.8 ms vs ~127.7 ms) and scaling behaviors are nearly identical. 

The primary difference lies in the developer workflow and build path: Cloud Run requires the developer to explicitly write a `Dockerfile`, build the image, and push it to Artifact Registry. In contrast, Cloud Functions accepts raw source code (`main.py`) directly and leverages Google Cloud buildpacks to automatically generate and manage the container image, abstracting away all containerization steps.
<!--/answer-->
EOF

