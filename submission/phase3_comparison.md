# Phase 3 — Comparison Writeup

## 1. Facts from your runs

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->36.0<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.009<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->5.1<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->5<!--/answer--> |

**Explain the timings.**

<!--answer:timing_explanation-->
From fastest to slowest: local single-process word count (`0.009 s`), local Spark TF-IDF (`5.1 s`), and cloud MapReduce word count (`36.0 s`). Only local vs. cloud word count is like-for-like; local execution runs entirely in memory, whereas cloud MapReduce spends almost all its time on HTTP invocations, Cloud Workflows scheduling, Cloud Storage I/O, and chaos retries. Spark computes a heavier TF-IDF pipeline (two `reduceByKey`s and a `join`) on one VM, paying JVM/task scheduling overhead once rather than per-task network round trips.
<!--/answer-->

**What the counts tell you.**

<!--answer:counts_interpretation-->
With 4 stages, the Spark job had **3 shuffle boundaries**, caused by `term_freq`'s `reduceByKey`, `doc_freq`'s `reduceByKey`, and `build_tfidf_rdd`'s `join` in `spark_tfidf.py`. Across 16 initial tasks (12 map + 4 reduce), 5 retries occurred (~31% of tasks, or 5/21 ≈ 24% of total attempts), closely matching the expected `--chaos 0.3` (30%) failure rate.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.**

<!--answer:transformations_vs_actions-->
In `spark_tfidf.py` and the notebook, `parallelize` creates the source RDD; `flatMap`, `map`, `reduceByKey`, and `join` are all lazy **transformations**, while `collect()` is the sole **action**. Calling `build_tfidf_rdd` returned immediately without processing data—it only recorded the DAG lineage graph. Calling `collect()` materialized the 4 stages, executed the 12 tasks across workers, and returned the 5,975 TF-IDF rows to the driver.
<!--/answer-->

**Lineage and recovery.**

<!--answer:lineage_recovery-->
If the worker holding one partition of the first `reduceByKey` (Stage 0/1 output) died, Spark would **not** need to rerun the downstream `doc_freq` or `join` stages from scratch—only the missing partition would be recomputed from its parent `ParallelCollectionRDD` partition via Stage 0 (`map`/`flatMap` + `reduceByKey` shuffle write boundary), while intact partitions and downstream shuffle files are reused.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (`workflow.yaml` / `mr-wordcount` workflow execution)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run Functions (`mr-worker` / `main.py` handling `map` and `reduce` requests)<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage (`clouds-lab-511413-eurecomgpt` bucket storing splits, intermediate shuffle objects, and `wordcount.json`)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->`partition_for` / `map_wc` in `phase-3-mapreduce-spark/mapreduce.py` (deterministic hash `zlib.crc32` modulo `num_reducers`)<!--/answer-->

**The barrier.**

<!--answer:barrier_location-->
The barrier is provided by the sequential workflow step structure in `workflow.yaml`: the parallel `for` loop (`parallel: for:`) running all map tasks must complete entirely before the workflow advances to the subsequent parallel `for` loop for the reduce tasks.
<!--/answer-->

**Idempotence.**

<!--answer:idempotence-->
Retries are safe because each map task writes deterministically to a fixed GCS object path (`shuffle/map-<split>-part-<r>.json`) using an **overwrite** (`upload_from_string`), replacing any partial or previous output cleanly. If `run_map` appended to the shuffle file instead, a retried task (or duplicate execution) would write its `(word, count)` pairs twice, inflating word counts in the reduce phase.
<!--/answer-->

**The chaos run.**

<!--answer:chaos_observations-->
With `--chaos 0.3`, the execution log and Logs Explorer showed 5 injected HTTP 503 transient failures (simulated worker unavailability) on individual map/reduce invocations, each automatically caught and retried by Cloud Workflows with exponential backoff. Despite those 5 retries, the workflow reached `SUCCEEDED` and produced `wordcount.json` with top terms identical to the local reference.
<!--/answer-->
