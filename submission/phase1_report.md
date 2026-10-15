# Phase 1 — Comparison Report

## 1. Setup facts

**Image size (MB, decimal):**

<!--answer:image_size_mb-->
49.987015
<!--/answer-->

Artifact Registry reports 49,987,015 bytes for `echo-bot:v1`. This is the registry-reported size, not the locally unpacked Docker size. The VM and Cloud Run use this same image; the source-built Function image was not measured here.

**Custom VPC and subnet:**

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

The `echo-http` network tag selects the VM to which the ingress rules apply; it is not an HTTP setting. The custom VPC has no default SSH allow rule. TCP 8080 allows public echo requests, while TCP 22 is restricted to the IAP forwarding range.

## 2. Measurements

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->237.0<!--/answer--> | <!--answer:vm_warm_median_ms-->211.2<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->3067.0<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->637.6<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1622.3<!--/answer--> | <!--answer:function_warm_median_ms-->637.2<!--/answer--> |

<!--answer:latency_plot-->
Warm-request histogram from the 20 recorded samples per platform. Each # represents one request; bins are lower-inclusive and upper-exclusive.

```text
Latency (ms)   VM                      Cloud Run        Function
[100, 200)                             ######## (8)     ###### (6)
[200, 300)     #################### (20)
[300, 400)     0                       0                0
[400, 500)     0                       0                0
[500, 600)     0                       0                0
[600, 700)                             ############ (12) ############## (14)
```

Blank cells mean zero. VM warm latency is tightly grouped at 210.4–215.1 ms. Both managed platforms have two clusters: approximately 120–135 ms and 635–688 ms. Their warm P95 values are 685.0 ms (Cloud Run) and 683.4 ms (Function), versus 213.7 ms for the VM. These are client-observed end-to-end measurements, not isolated application execution times. The data alone cannot establish the cause of the two clusters.
<!--/answer-->

The harness labels the first request `cold_ms` but does not force or verify scale-to-zero. The VM container remained running, so its 237.0 ms is not a container cold start. A single first-request sample per platform is insufficient to estimate a cold-start distribution.

## 3. Per-platform analysis

**Scaling behaviour:**

<!--answer:scaling_behaviour-->
For sequential requests, the VM uses its existing container; bursts of ten or a hundred create nothing automatically. Capacity is constrained by the e2-micro CPU/memory and Gunicorn's one worker with eight threads. Cloud Run and the gen-2 Function can start additional container instances as demand increases, with configured minimum zero and maximum three instances. Ten or a hundred requests do not imply ten or a hundred instances: concurrency and capacity determine scaling. At saturation, requests can queue or fail. These are expected behaviours, not measured load-test results.
<!--/answer-->

**Deployment effort:**

<!--answer:deployment_effort-->
All paths share the echo logic and unit tests. The VM required a custom VPC, subnet, tagged firewall rules, service-account image-read permissions, a Debian e2-micro, SSH access, Docker installation, registry authentication and container startup. I maintain its OS and container lifecycle. Cloud Run reused the same image with public access, port 8080 and scaling limits, without managing a VM. Direct Docker push failed with connection refused; Cloud Build provided the image-build/upload path. The Function deployed main.py and requirements.txt with Python 3.11 and entry point echo; Google built its runtime image. Both managed paths still require IAM, logs and deployment configuration.
<!--/answer-->

## 4. Why do two of your three numbers look alike?

Output of `gcloud run services list --project=eurecomgpt-511407 --region=us-central1`, as captured in Cloud Shell:

<!--answer:run_services_list-->
```text
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-1038300781967.us-central1.run.app
LAST DEPLOYED BY: service-1038300781967@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-15T20:38:23.665216Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-1038300781967.us-central1.run.app
LAST DEPLOYED BY: Xuanhui.Yang@eurecom.fr
LAST DEPLOYED AT: 2026-10-15T20:36:32.403256Z
```
<!--/answer-->

<!--answer:convergence_explanation-->
The list contains both echo-bot and echo. The Function service echo was deployed by the Cloud Functions service agent, demonstrating its Cloud Run backing. Both therefore use managed container instances and can incur startup overhead after scaling to zero. Their warm medians are almost identical: 637.6 versus 637.2 ms. Their first-request times, however, differ substantially: 3067.0 versus 1622.3 ms; shared infrastructure does not imply identical startup latency. Cloud Run used our Dockerfile-built image and Gunicorn command, whereas the Function accepted source and dependencies, with Google building the image around Functions Framework and entry point echo. This changes packaging and runtime setup while retaining the same echo logic. The measurements support similar warm behaviour, not a proven cause for every latency difference.
<!--/answer-->

Architecture reference: [Google Cloud function deployment resource model](https://docs.cloud.google.com/run/docs/resource-model).

Validation evidence: the supplied Cloud Shell test output reports **13 passed, 85/85**. GitHub Actions success and resource teardown remain separate steps.