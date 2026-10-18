# Phase 1 Comparison Report: IaaS, PaaS, and Serverless

## 1. Executive Summary & Overview
This report analyzes the deployment, performance, and operational overhead of hosting an echo-bot application across three distinct cloud paradigms on Google Cloud Platform: an Infrastructure-as-an-Service (IaaS) virtual machine, a Containerized Platform-as-a-Service (Cloud Run), and a Serverless Function (Cloud Functions Gen 2).

---

## 2. Platform Comparison Table

| Metric / Aspect | IaaS (Compute Engine VM) | PaaS (Cloud Run) | Serverless (Cloud Functions Gen 2) |
| :--- | :--- | :--- | :--- |
| **Deployment Model** | Manual OS, Docker daemon, custom VPC & Firewall | Container image pushed to Artifact Registry | Direct source code deployment (`main.py`) |
| **Image Size / Build** | Base Debian + manual installation | ~50-100 MB (via Dockerfile) | Managed automatically by buildpacks |
| **Cold Start Latency** | ~434.6 ms (Persistent instance) | ~1613.3 ms | ~785.0 ms |
| **Warm Median Latency** | ~212.1 ms | ~127.7 ms | ~127.8 ms |
| **Operational Overhead** | High (VPC, firewall tags, SSH, updates) | Low (Managed container runtime) | Minimal (Function code only) |

---

## 3. Detailed Answers & Question Responses

### Question 1: Latency Analysis & Histograms
<!--answer:cold-warm-latencies-->
Measured values from `phase1_report.json`:
- **VM (`vm`)**: Cold = 434.6 ms, Warm Median = 212.1 ms. The VM exhibits a very minimal difference between cold and warm requests because the compute instance and its container run continuously without scaling down to zero.
- **Cloud Run (`cloudrun`)**: Cold = 1613.3 ms, Warm Median = 127.7 ms. The cold start includes container provisioning and initialization overhead when scaling from zero, while warm requests execute rapidly.
- **Cloud Functions (`function`)**: Cold = 785.0 ms, Warm Median = 127.8 ms. The cold start is moderately fast, and warm latency matches Cloud Run perfectly due to the shared underlying architecture.
<!--/answer-->

### Question 2: Deployment Effort & Configuration Complexity
<!--answer:deployment-effort-->
- **VM**: Requires manual infrastructure provisioning: creating a custom VPC (`echo-net`), subnets, ingress firewall rules (TCP 8080 and TCP 22 for SSH), configuring IAM roles for Artifact Registry access, installing Docker inside the OS, and running the container manually.
- **Cloud Run**: Simplifies operations by abstracting infrastructure; requires only a valid `Dockerfile` and a single `gcloud run deploy` command targeting Artifact Registry.
- **Cloud Functions**: The simplest deployment path, requiring only raw source code (`main.py` and `requirements.txt`). Google Cloud builds and manages the container automatically via buildpacks without requiring any Docker knowledge.
<!--/answer-->

### Question 3: Scaling Behavior & Cost Implications
<!--answer:scaling-behavior-->
- **VM**: Runs 24/7 with `min-instances=1` (or persistent running), incurring continuous costs for the external IP address even when idle. Does not scale automatically to zero out-of-the-box.
- **Cloud Run & Cloud Functions**: Both scale dynamically from 0 to 3 (or more) instances based on incoming traffic and scale back down to zero after a period of inactivity (~15 minutes). This ensures true pay-per-use economics and zero cost when idle.
<!--/answer-->

### Question 4: Relationship Between Cloud Run and Cloud Functions (The Core Analysis)
<!--answer:cloud-run-functions-relationship-->
When executing `gcloud run services list`, the underlying service created for the Cloud Function is visible directly within Cloud Run. Under the hood, Cloud Functions Gen 2 runs directly on top of Cloud Run infrastructure. 

This explains why their warm latencies (~127.8 ms vs ~127.7 ms) and scaling behaviors are nearly identical. The primary difference lies in the developer experience: Cloud Run requires explicit containerization (`Dockerfile`), whereas Cloud Functions manages the build process automatically from raw source files (`main.py`).
<!--/answer-->
