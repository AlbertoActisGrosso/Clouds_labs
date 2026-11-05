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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->264.172<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.202<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->458741.9<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12935.9<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4274<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
"1": 2481.241,
"2": 1991.445,
"4": 2665.508,
"8": 2901.134
Fastest : 2 threads
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive implementation spends the vast majority of its execution time handling Python interpreter overhead, dynamic type checking, and scalar instruction dispatch inside nested loops. In contrast, the vectorized NumPy implementation bypasses Python by delegating array operations directly to C/BLAS libraries (OpenBLAS). At the machine level, it leverages CPU SIMD registers (AVX instructions) to process multiple floating-point values per clock cycle while maintaining contiguous memory access for optimal cache usage.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Performance peaked at 2 threads (1991.445 ms) and degraded significantly at 4 threads (2665.508 ms) and 8 threads (2901.134 ms). The Google Colab runtime allocates only 2 physical vCPUs. Exceeding 2 threads forces CPU over-subscription, causing severe thread contention, context-switching overhead, and synchronization delays. Furthermore, multi-threading reaches a memory bandwidth bottleneck (Amdahl's Law), where thread coordination on shared tensor memory costs more than the parallel computation saves.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Transformer training steps consist mostly of massive matrix multiplications (such as query-key dot products, projections, and feed-forward layers), which are highly parallelizable independent computations perfectly suited for the GPU SIMT architecture and thousands of CUDA cores. Conversely, a workload with extremely small batch sizes (e.g., batch size 1 during sequential auto-regressive generation) would not benefit, as GPU compute units remain underutilized while memory bandwidth and PCIe host-to-device latency dominate the runtime.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when constrained by hardware limits, coordination overhead, or memory bottlenecks, as illustrated by Amdahl's Law.

First, vectorization eliminated Python interpreter overhead by utilizing hardware SIMD units, boosting execution from 264.172 ms to 1.202 ms (219.78x speedup).

Second, thread parallelism hit a hard physical limit: scaling from 1 thread (2481.241 ms) to 2 threads (1991.445 ms) improved speed, but 4 threads (2665.508 ms) degraded performance because Colab only has 2 physical vCPUs—making context switching and thread coordination cost more than the compute saves.

Third, GPU acceleration delivered a massive gain over CPU (12935.9 ms vs 458741.9 ms for 100 steps, a 35.5x speedup) by parallelizing matrix operations across thousands of CUDA cores. However, GPU scaling also stops helping when workloads fail to saturate the cores or become memory-bandwidth bound during host-to-device transfer.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would sweep batch sizes on the GPU (e.g., batch sizes 1 to 128) to measure exact GPU core saturation and kernel launch overhead. Finding that performance per sample stays flat at tiny batch sizes would confirm that memory transfer and kernel launch latency bound small workloads.
<!--/answer-->
