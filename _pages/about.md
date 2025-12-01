---
layout: about
title: proposal
permalink: /

profile:
  align: right
  image:
  image_circular: false # crops the image to make it circular
  more_info:

news: false # includes a list of news items
selected_papers: false # includes a list of papers marked as "selected={true}"
social: false # includes social icons at the bottom of the page
---

Created by Carolyn Liu (cyl2) and Jessica Qiu (jessicaq)

**Project Proposal:** [link](https://docs.google.com/document/d/12OziUmKmf31YXhJEWzywiWQOCklERNBT52Fa94qL2N4/edit?usp=sharing)

### **Summary**

We will develop a parallel Othello solver using the Monte Carlo Tree Search algorithm and analyze its performance under two different parallelization strategies. One approach will involve multi-threading on the CPU for root parallelism to run independent searches from the same root state and merge the root results at the end. The second approach uses leaf parallelism, where CUDA runs many simulated games in parallel from newly expanded leaf nodes on the GPU, while the CPU handles the shared tree.

### **Background**

The Monte Carlo Tree Search algorithm (MCTS) is a heuristic search algorithm most notably used to play games. It emerged in the mid-2000s as a breakthrough in game-playing AI: before MCTS, complex games such as Go were extremely difficult because traditional search couldn’t handle the enormous branching factor. In 2016, it was combined with neural networks by Google Deepmind to create the computer programs AlphaGo, AlphaGo Zero, and AlphaZero, which were the first programs that could beat a professional human player without handicaps in a standard game.

The core idea of MCTS is to sample the space by repeatedly simulating games to estimate how good each move is, rather than exhaustively searching the whole game tree. The basic algorithm has four steps, with different opportunities for parallelism as follows:

1. Selection: from the current game state (the root), choose different moves to traverse down the game tree until you reach a leaf node, an unexplored state.
   1. Multiple independent root parallel searches can be run concurrently on CPU threads, each building a separate tree from the same root.
2. Expansion: in the unexplored state, if the game is not yet ended, add child nodes corresponding to possible moves from the game state in the leaf node. Choose one new child node _C_.
   1. This is generally lightweight, but can still be executed concurrently on multiple CPU threads if leaf nodes are expanded in a batch.
3. Simulation: from the new node _C_, complete a full simulation of the game; that is, play the game out until the very end. Moves can be chosen as desired.
   1. Because each rollout is independent, multiple rollouts can be executed in parallel on the GPU, either across leaf nodes or across multiple simulations of the same leaf.
4. Backpropagation: use the result of the played out game to update the information in the nodes from the root to _C_.
   1. When running multiple independent trees for root parallelism, backpropagation can occur simultaneously for each tree on separate CPU threads.

The most basic estimation is to choose the move that leads to the most victories. Over many simulations, the tree grows asymmetrically: more effort is spent on promising playouts, not on exhaustive or uniform search.

### **Challenge**

To parallelize MCTS, we will explore and compare 2 different approaches, each with their own tradeoffs.

MCTS is inherently difficult to parallelize due to its sequential dependencies. Stages like selection, expansion, and backpropagation depend on up-to-date tree statistics, and the branches may expand at different rates. The game simulations also vary in length and branching can create irregular execution patterns and divergent control flow, which are challenging for SIMD-style hardware like GPUs. Also, tree accesses are often noncontiguous, limiting memory locality, and moving leaf states to the GPU can increase communication overhead and thus the communication to computation ratio as well. These factors make efficient parallel mapping difficult, but throughout this project, we hope to be able to learn how to parallelize MCTS in a way to minimize synchronization and maximize throughput.

First, we will explore root parallelism on the CPU with multi-threading. Here, multiple workers execute independent searches from the same root state, each building their own tree. Since the trees are independent, no locking is needed. At the end, root level statistics like visit counts are aggregated to choose the best move. This strategy is simpler to implement and scales well across CPU threads, but may have higher memory costs due to duplicated trees. Also, workers may redundantly explore similar regions of the tree, reducing efficiency.

Next, we will implement leaf parallelism using CUDA. In this approach, the CPU handles the shared tree for selection, expansion, and backpropagation, while the independent game simulations (rollouts) from newly expanded leaf nodes are performed by the GPU in parallel. Since the rollouts are independent and uniform, the GPU can be effectively utilized, avoiding tree synchronization issues and allowing for higher throughput. However, there may be data transfer bottlenecks due to transferring batches of leaf states to the GPU and backpropagation of the results.

### **Resources**

We will reference the following sequential implementations of the MCTS algorithm for our project:

- MCTS algorithm tutorial: [https://github.com/ai-boson/mcts](https://github.com/ai-boson/mcts)
- Reversi Monte Carlo Tree Search: [https://gitlab.com/royhung\_/reversi-monte-carlo-tree-search](https://gitlab.com/royhung_/reversi-monte-carlo-tree-search)
- Reversi AI experiments: [https://github.com/psaikko/mcts-reversi](https://github.com/psaikko/mcts-reversi)

As well as the following research papers on parallelizing MCTS for other games:

- MCTS GPU parallelization in Python: [https://www.sciencedirect.com/science/article/pii/S2352711025001062](https://www.sciencedirect.com/science/article/pii/S2352711025001062)
- Parallelizing MCTS: [https://www-users.cse.umn.edu/\~gini/publications/papers/Steinmetz2020TG.pdf](https://www-users.cse.umn.edu/~gini/publications/papers/Steinmetz2020TG.pdf)

We plan to use the CPUs and GPUs on the Gates cluster machines to implement, test, and analyze our project, as they are readily available and already have CUDA and OpenMP installed.

### **Goals and deliverables**

Plan to achieve:

- Implement the MCTS sequential algorithm targeting Othello
- Parallelize MCTS with the 2 parallelization strategies (see Challenge section)
  - Root parallelism on the CPU
  - Leaf parallelism with CUDA
- Develop benchmark scripts to evaluate both approaches on Othello, using metrics like throughput and search depth
- Provide an analytic comparison between the 2 approaches including speedup, memory usage, and sample efficiency

Hope to achieve:

- Explore a hybrid approach with both CPU root parallelism and GPU leaf rollouts, aiming for both high throughput and good move selections
- Test scalability of GPU leaf parallelism on large Othello board inputs

### **Platform choice**

The hardware platform for our project includes both an Intel i7-9700 CPU and NVIDIA GPU (RTX 2080\) on the GHC cluster machines. For root parallelism, the CPU is the primary platform, as each worker builds independent MCTS trees in parallelism, which doesn’t require synchronization. CPUs are well suited for this approach because tree operations involve irregular memory accesses and branching, which are handled poorly on the GPU.

For leaf parallelism, the GPU is the primary platform. The workload of rollouts is massively parallel; each rollout follows the same algorithm and threads can execute the same instructions so it’s uniform and independent. Thus, there’s also minimal synchronization and predictable memory access, so this is the type of computation the GPU is designed to accelerate. However, the CPU still manages the shared tree for the selection, expansion, and backpropagation stages to avoid complex synchronization on the tree.

### **Schedule**

|                             | Carolyn                                                                                                                                                                                                | Jessica                                                                                                                                                                                                    |
| :-------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Week 1:** Nov 17 – Nov 23 | Implement basic Othello game logic Implement the sequential MCTS algorithm for Othello with UCT                                                                                                        | Set up CUDA project structure. Identify data structures requiring CPU–GPU synchronization. Begin early GPU prototype for fast rollouts (simple random playout kernel).                                     |
| **Week 2:** Nov 24 – Nov 30 | Implement multi-threading for root parallelism using OpenMP. Develop initial benchmarking script (throughput, rollout count, tree size).                                                               | Implement GPU rollout kernel launching from new leaf nodes. Design CPU-GPU communication patterns                                                                                                          |
| **Week** Nov 31 – Dec 7 | Finalize benchmarking scripts and statistics Fine-tune and finalize root parallelism based on the benchmark scripts Run experiments on root parallelism with different problem sizes and analyze the final results | See if GPU version is feasible still, polish CPU version of leaf parallelism Fine-tune and finalize leaf parallelism based on the benchmark scripts Run experiments on leaf parallelism with different problem sizes and analyze the final results |

**50% Milestone:**  
Working sequential version of pure MCTS Othello solver and initial benchmark scripts to evaluate the performance of an Othello solver. GPU development environment ready; simple rollout kernel functional.

**75% Milestone:**  
Root parallelism works with multi-threading; can run full games. Leaf parallelism can run batches of GPU rollouts triggered from leaf nodes. Basic performance comparisons possible on small boards.

**100% Milestone:**  
Fully functioning Othello solver with sequential MCTS, CPU root parallelism, and GPU leaf parallelism.  
Benchmark suite completed (throughput, search depth, speedup). Full analysis report including memory usage and sample efficiency.

**125% Milestone:**  
Implement the hybrid MCTS approach (CPU root parallelism combined with GPU leaf rollouts). Scalability tests on larger Othello boards.

**150% Milestone:**  
Adaptive parallel MCTS strategy: dynamically choose CPU vs GPU rollouts based on tree shape, and optimize implementations.
