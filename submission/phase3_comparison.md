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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->41.6<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.012<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.9<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->7<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranking from fastest to slowest: local single-process (0.012 s), Spark (4.9 s), and cloud MapReduce (41.6 s). 
Only local vs. cloud MapReduce is like-for-like (both run word count). Local runs entirely in RAM with zero serialization or network overhead. Spark executes a heavier multi-stage TF-IDF pipeline, spending its time on task scheduling, Python worker serialization and local shuffles. Seeing Spark finish a harder job in 4.9 s proves that cloud MapReduce's 41.6 s is dominated by distributed I/O overhead: HTTP invocations, GCS object reads/writes, workflow scheduling and retry backoff delays rather than computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
Four stages correspond to three shuffle boundaries (4 - 1 = 3), triggered by reduceByKey in term_freq, reduceByKey in doc_freq, and join in build_tfidf_rdd. With 7 retries across 16 base tasks (23 total attempts), about 30.4% (7/23) of attempts failed and retried, matching --chaos 0.3 (30% failure rate).
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
Parallelize, flatMap, reduceByKey, map, and join are all transformations; collect is the sole action. Calling build_tfidf_rdd executed zero computation—it lazily constructed and recorded the RDD lineage DAG in memory. Only when collect() was invoked did Spark materialize the DAG into stages, launch tasks, execute the shuffles and return the 5,975 rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Spark would recompute only the single lost partition starting from the source `ParallelCollectionRDD[0]` through Stage 0 (`flatMap` and the first `reduceByKey` up to the `ShuffledRDD[3]` boundary). Any surviving sibling partitions of `ShuffledRDD[3]` on other workers are reused rather than restarting the entire job from scratch.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (workflow.yaml / mr-wordcount)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run functions (2nd gen, mr-worker / main.py)<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition(word, num_reducers) in mapreduce.py (using zlib.crc32(word) % num_reducers)<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is provided by Cloud Workflows' sequential step execution immediately following the parallel for block in map_phase: reduce_phase is the next top-level step and cannot begin until every parallel map branch completes.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each mapper writes its output to a deterministic GCS object path keyed by (job, split, reducer) and overwrites it atomically upon completion. Retrying a failed attempt simply replaces any partial/stale file with identical data, making map tasks idempotent. If run_map appended instead, a task crashing mid-write and retrying would duplicate already-emitted (word, 1) pairs, inflating final word counts.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
In Logs Explorer and the Workflows execution log, 7 worker invocations failed with HTTP 503 errors and triggered automatic exponential-backoff retries. Despite the injected failures, the workflow recovered all tasks, finished with state SUCCEEDED in 41.6 s, and produced output identical to the local reference (matches_local: true).
<!--/answer-->
