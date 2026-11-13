# Phase 3 — Comparison Writeup

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->38.5<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.009<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->3.9<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->8<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Local word count was fastest (0.009 s), followed by Spark TF-IDF collect (3.9 s), then cloud word count with chaos enabled (38.5 s). Local and cloud word count perform the same computation, although on different environments; Spark computes the heavier TF-IDF pipeline. Local execution avoids network overhead. Spark pays task scheduling, serialization and shuffle costs, but its collect timer excludes earlier SparkSession startup. Cloud execution additionally pays function invocations, GCS round trips, workflow orchestration and retry delays. This small corpus cannot amortize that overhead.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
Four stages span three distinct shuffles: TF reduceByKey, DF reduceByKey and join. There were 16 logical tasks and 8 extra attempts (8/16 = 50% retry overhead), not 16 total attempts. Logs show 7 HTTP 503s, 1 HTTP 429 and 16 successes: 24 attempts. Excluding the 429, 7/23 = 30.4% were 503s, consistent with chaos=0.3 in this small sample. The number of distinct retried tasks is unknown.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
flatMap emits token pairs; map reshapes keys or constructs rows; reduceByKey aggregates TF and DF; join combines them. These are lazy transformations. build_tfidf_rdd constructed dependencies without executing the TF-IDF job, and toDebugString displayed them. collect triggered execution of four stages and returned 5975 rows to the driver. parallelize created the input RDD; the subsequent save_parquet used PyArrow on the collected rows, not another Spark action.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Recovery starts at the first TF shuffle, ShuffledRDD[3], between stages 0 and 1. Spark can recompute a lost TF partition from surviving stage-0 shuffle blocks, reusing unaffected outputs. If the failed node also lost those blocks, their producing map tasks must be rerun from the input. Downstream affected partitions are recomputed as needed; the entire job need not restart. This is a recovery analysis, not an observed Spark failure.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows service mr-wordcount, defined in workflow.yaml.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Invocations of the private mr-worker Cloud Run function, dispatched by main.py to map or reduce tasks.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage bucket eurecomgpt-511407-eurecomgpt and its per-job input/, shuffle/ and output/ prefixes; object storage serves the storage role, rather than being actual HDFS.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->mapreduce.py: partition(word, num_reducers), using stable zlib.crc32(word.encode('utf-8')) modulo the reducer count.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The map_phase parallel step completes only after all its branches finish successfully. The next sequential step, reduce_phase, therefore cannot start until all mapper outputs have been written; no separate wait instruction is needed.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each task reads fixed inputs and overwrites deterministic, job-specific output paths. Repeating it produces the same logical contents, even if an earlier attempt wrote some outputs before failing. Appending mapper pairs instead would duplicate already-written tokens and inflate reducer counts. The map barrier prevents reducers from reading incomplete map output. The chaos experiment only injected failures before work, so it did not directly test partial-write recovery.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
The console execution page failed to load, so I used gcloud executions describe and Cloud Run request logs instead of claiming a visual inspection. In the execution interval, logs contained 16 HTTP 200s, 7 HTTP 503s and 1 HTTP 429. Assuming this interval contained only this job and complete request logs, these imply 8 retries. The job SUCCEEDED with 2221 distinct words and matches_local=true. The 503s are consistent with simulated crashes; the 429 is a separate capacity/rate-limit response.
<!--/answer-->

## Recorded execution evidence

- Workflow execution: `278c40db-08c0-4d81-b1e4-da3e2d2f2b12`; job: `mr-20261113-192514`.
- Workflow service duration: 36.781662117 s; client report duration: 38.5 s. The table uses the required report value, not the service-only duration.
- Final report local timing: 0.009 s from Colab, replacing the original Cloud Shell reference timing of 0.01 s.
- Spark job 0: 4 stages, 12 tasks. Stages 0–3 had 2, 2, 4 and 4 tasks. Shuffle writes were 51.0, 23.7, 65.4 and 0.0 KB; reads were 0.0, 51.0, 74.7 and 65.4 KB (display-rounded values).
- Spark UI window/iframe display did not work in Colab. Stage counts and shuffle metrics came from notebook Section 3a, not an unobserved UI screenshot.
- Parquet output: 5975 rows. BigQuery table: `eurecomgpt-511407.eurecomgpt.tfidf`. Highest reported TF-IDF result: `voices`, document 56, 17.9176.
