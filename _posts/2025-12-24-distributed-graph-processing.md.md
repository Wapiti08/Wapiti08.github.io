---
layout: post
title: "Distributed Graph Processing in Practice: From Python Parallelism to Rust and Go"
date: 2025-12-24
tags: [graph-processing, distributed-systems, rust, golang, numba, joblib]
---

## Introduction

Large-scale graph processing becomes difficult not because BFS or reachability analysis is conceptually complex, but because real-world graphs quickly expose bottlenecks in memory layout, synchronization, scheduling, and data movement.

This post summarizes my practical exploration of several implementations, starting from Python-based parallelism with Numba and Joblib, then moving toward Rust and Go for stronger control over memory and concurrency.

## Problem Context: Attack Chain Analysis
In security analytics, we often need to trace propagation paths through system dependency graphs. A typical scenario involves:
- Node: system entities (processes, files, network connections)
- Edges: dependencies or information flows
- Graph size: 10⁶ to 10⁸ nodes, 10⁷ to 10⁹ edges
- Query pattern: Breadth-first search (BFS) or reachability analysis from multiple seed nodes
- Constraint: real-time or near-real-time response requirements

The challenge is to parallelize graph traversal while managing:
1. Load balancing across workers
2. Memory overhead from duplicate work
3. Synchronization costs
4. Cache efficiency

Although the examples are motivated by attack-chain analysis, the same pattern appears in dependency analysis, provenance tracking, fraud detection, and large-scale system telemetry.

## Approach 1: Python with Numba JIT (just-in-time)

This first optimization direction was to keep the implementation in Python while moving the hottest traversal loops closer to native execution. Numba is attractive in this setting because it allows numerical Python code to be compiled just-in-time, often with minimal changes to the surrounding workflow. JIT compilation offers significantly higher execution efficiency compared to interpreted languages like Python.

However, Numba is most effective when the graph is represented in an array-oriented format. A Python dictionary of lists is convenient for prototyping, but it still carries significant object overhead. For graph traversal, a more suitable representation is usually CSR (Compressed Sparse Row), where adjacency information is stored in compact arrays.

```python
@njit
def bfs_csr(indptr: np.ndarray, indices: np.ndarray, source: int):
    '''
    args:
        indptr: index pointer, row offset array
        indices: column indices. column index array
        source: starting node index
    '''
    num_nodes = len(indpter) - 1
    visited = np.zeros(num_nodes, dtype=np.bool_)
    queue = 



```