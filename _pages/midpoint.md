---
layout: about
title: midpoint
permalink: /midpoint/

profile:
  align: right
  image:
  image_circular: false # crops the image to make it circular
  more_info:

news: false # includes a list of news items
selected_papers: false # includes a list of papers marked as "selected={true}"
social: false # includes social icons at the bottom of the page
---

**Midpoint Report:** [link](https://docs.google.com/document/d/1S-EgAtiITfOTKMkug9DiR-2xY8l65mbl-evFzErILUE/edit?usp=sharing)

### Milestone Progress

We have mostly maintained our schedule with a few hiccups along the way. We finished our sequential version of the Othello solver using MCTS, and produced an interactive game against the solver along with some testing for correctness. For example, we varied simulation counts and measured wins against a player that makes random moves.

We also completed a rough implementation of root parallelism using OpenMP, and achieved promising initial results for correctness and speedup of the implementation. After further research on potential opportunities for parallelization and optimization, we also experimented with different strategies for root parallelization. On top of just exploring the search space in parallel, we also tried utilizing virtual loss to encourage threads to explore different parts of the tree. This strategy is when a thread temporarily simulates that a node has lost a game, making it less attractive for other threads for the UCT formula. This ideally prevents multiple threads from simulating the same node at the same time, reducing contention and making exploration more diverse.

For leaf parallelism, the original idea was to implement a rollout kernel on the GPU with CUDA. However, as explained in Goals and Deliverables below, we struggled to make this work, so we opted for a CPU version instead using OpenMP, as this would ensure a similar environment to root parallelism, giving a more accurate comparison between the two.

To test the performance and correctness of our parallel versions, we added benchmarking to time the games for each version and measure the speedups against the baseline sequential version. We experimented with different parameters such as varying thread sizes and simulation counts. So far the correctness of all of the versions were around the same, and the speedups of the 2 parallel versions were comparable (both around 6x).

### Updated Schedule

|                         | Carolyn                                                                                                                                                                                                            | Jessica                                                                                                                                                                                                                                            |
| :---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Week** Nov 31 – Dec 7 | Finalize benchmarking scripts and statistics Fine-tune and finalize root parallelism based on the benchmark scripts Run experiments on root parallelism with different problem sizes and analyze the final results | See if GPU version is feasible still, polish CPU version of leaf parallelism Fine-tune and finalize leaf parallelism based on the benchmark scripts Run experiments on leaf parallelism with different problem sizes and analyze the final results |

### Goals and Deliverables

​​  
We have not quite achieved our goal of having a GPU version of leaf parallelism, because through experimenting with it we found that our speedups were very small. We hypothesized that this could be because MCTS expands the tree progressively, so early in the search there are only a few leaves, and each leaf receives only a small number of rollouts. This results in very small GPU kernels that fail to fully utilize the hardware. Our simulation logic is relatively complex and branch-heavy, causing each GPU thread to do mostly sequential work and introducing warp divergence, which could further reduce parallel efficiency. We attempted to simplify the code by just choosing random moves instead to reduce the amount of sequential work, but while this improved our speedups a little bit, it was still low compared to our CPU versions and furthermore, our wins were less. On the CPU, OpenMP allows multiple threads to efficiently handle the branchy, irregular simulation logic with minimal overhead, providing better parallel performance and more consistent win rates. As a result, we migrated our leaf-parallel implementation to the CPU using OpenMP. We got decent speedups of around 6x. However, we will still experiment more with the CUDA version in the next week to see if we can get better speedups while not losing out on correctness

We have not yet finalized our implementation of root parallelism and plan to experiment further with both approaches to improve speed and performance. In the basic version, each thread clones the entire root tree and runs MCTS iterations independently, after which thread-local statistics are merged back into the main root. Although simpler, this approach incurs high memory usage and merging overhead. As such, in the more optimized version with virtual loss, threads share the same root tree and traverse it concurrently using atomic updates and locks. However, this version currently performs slightly below the basic version, although its win rate is slightly better. Thus, we plan to fine-tune the implementation to identify bottlenecks and improve performance.

#### **Preliminary results**

These results were from experiments run on the GHC machines.

**Parallelization Performance vs Random (2000 Simulations, 40 Games, 8 Threads)**

| Strategy                         | Win Rate | Average Time/Move (s) | Total Time (s) | Speedup |
| -------------------------------- | -------- | --------------------- | -------------- | ------- |
| **Sequential**                   | 100.0%   | 0.0516                | 61.47          | 1.00x   |
| **Leaf Parallel**                | 100.0%   | 0.0088                | 10.47          | 5.87x   |
| **Root Parallel**                | 100.0%   | 0.0085                | 10.35          | 5.94x   |
| **Root Parallel (Virtual Loss)** | 100.0%   | 0.0096                | 11.67          | 5.27x   |

**Thread Scaling vs Random (2000 Simulations, 20 Games)**  
Leaf Parallel:

| Number of Threads | Average Time/Move (s) | Total Time (s) | Speedup |
| ----------------- | --------------------- | -------------- | ------- |
| **1**             | 0.0510                | 61.20          | 1.00x   |
| **2**             | 0.0264                | 31.40          | 1.93x   |
| **4**             | 0.0141                | 16.93          | 3.62x   |
| **8**             | 0.0102                | 12.27          | 4.99x   |

Root Parallel:

| Number of Threads | Average Time/Move (s) | Total Time (s) | Speedup |
| ----------------- | --------------------- | -------------- | ------- |
| **1**             | 0.0510                | 61.18          | 1.00x   |
| **2**             | 0.0265                | 31.81          | 1.92x   |
| **4**             | 0.0142                | 17.09          | 3.58x   |
| **8**             | 0.0092                | 11.00          | 5.56x   |

Root Parallel (Virtual Loss):

| Number of Threads | Average Time/Move (s) | Total Time (s) | Speedup |
| ----------------- | --------------------- | -------------- | ------- |
| **1**             | 0.0538                | 64.56          | 1.00x   |
| **2**             | 0.0278                | 33.38          | 1.93x   |
| **4**             | 0.0151                | 18.13          | 3.56x   |
| **8**             | 0.0098                | 11.80          | 5.47x   |

**Simulation Scaling Win Count vs Random (2000 Simulations, 15 Games, 8 Threads)**:

| Number of Simulations | Sequential | Leaf Parallel | Root Parallel | Root Parallel (Virtual Loss) |
| --------------------- | ---------- | ------------- | ------------- | ---------------------------- |
| 500                   | 15/15      | 13/15         | 15/15         | 15/15                        |
| 1000                  | 15/15      | 14/15         | 15/15         | 15/15                        |
| 2000                  | 15/15      | 13/15         | 13/15         | 15/15                        |
| 2500                  | 14/15      | 15/15         | 15/15         | 15/15                        |

### Concerns

Our current concern is that our GPU version of leaf parallelism might not be feasible, so we can’t achieve good speedups without heavily bringing down the solver’s correctness. However, we plan to continue optimizing our implementation while preserving correctness, by exploring strategies such as reducing CPU–GPU transfer overhead and minimizing warp divergence. If we still can’t produce good results, we will shift our focus to the two CPU versions and continue optimizing those.
