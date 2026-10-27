# Phase 2 — Reflection: where does parallelism stop helping?

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->797.956<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->19.948<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->451451.1<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13387.3<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4971<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
2419.031, 2088.885, 2305.2, 2440.896 ms for 1, 2, 4 and 8 threads, respectively. Two threads were fastest. The runtime reported 2 logical CPUs, with 2 CPUs available to the process.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Naive attention took 797.956 ms because Python loops repeatedly dispatch scalar operations and index arrays rather than keeping arithmetic in an optimized native kernel. Vectorization reduced this to 19.948 ms, a 40.0x speedup. NumPy delegates matrix operations to OpenBLAS, which can exploit SIMD instructions, cache-friendly data reuse and optimized loops. The configuration detected AVX and AVX2 support, but this does not prove which instructions executed or how much each optimization contributed.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The runtime exposed 2 logical CPUs, both available to the process. Two threads were fastest at 2088.885 ms, versus 2419.031 ms with one. Four and eight threads took 2305.2 and 2440.896 ms: adding threads beyond two hurt performance. Oversubscribed workers compete for CPU time and coordinate access to shared data and operation completion. Scheduling, synchronization and cache or memory contention are plausible contributors; these timings do not isolate their individual costs. The sweep restored PyTorch's original one-thread setting afterward.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
The 100-step training benchmark fell from 451451.1 ms on CPU to 13387.3 ms on CUDA, about 33.72x faster. Transformer matrix multiplications and their gradients expose many independent arithmetic operations, allowing GPU execution units to process large batches in parallel. This amortizes kernel-launch and data-transfer overhead. A tiny, sequential, branch-heavy workload would not necessarily benefit: too little parallel work could leave the GPU underused while those overheads dominate.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when extra execution resources no longer offset overhead or cannot accelerate the remaining work. Attention improved from 797.956 to 19.948 ms by replacing interpreter-driven loops with optimized array operations; this demonstrates avoidable overhead, not that SIMD alone explains all 40.0x. Residual serial work and data movement impose limits, but this experiment does not identify the remaining bottleneck. CPU threading shows a clearer resource limit: only two logical CPUs were available, and two threads achieved 2088.885 ms, while eight needed 2440.896 ms. More software workers did not create more hardware; coordination and contention could outweigh gains. GPU training reached 13387.3 ms versus 451451.1 ms on CPU because the workload offered substantial parallel arithmetic. Nevertheless, sequential dependencies, host work, kernel launches and memory transfers cannot all be eliminated. The CPU baseline used the restored one-thread setting, so 33.72x is not a comparison against the best tuned CPU configuration.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would repeat the 100-step CPU/GPU benchmark across batch sizes, using the fastest measured two-thread CPU setting, GPU warm-up, CUDA synchronization and repeated trials. I expect smaller batches to reduce GPU advantage. If the speedup remains unchanged, launch and transfer overhead would be less convincing explanations for that workload.
<!--/answer-->
