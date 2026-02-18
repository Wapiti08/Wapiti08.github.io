---
layout: post
title: "Distributed Graph Processing in Practice: From Python Parallelism to Rust and Go"
date: 2025-12-24
tags: [graph-processing, distributed-systems, rust, golang, numba, joblib]
---

## Introduction

Large-scale graph processing is a recurring challenge in both security analytics
and systems research. In my recent work on dependency graphs and attack-chain
analysis, I experimented with multiple parallel and distributed execution models
to scale graph computation efficiently.

This post reflects on a comparative exploration of several approaches:
earlier Python-based solutions using **Numba** and **Joblib**, and more recent
systems-level implementations in **Rust** and **Go**.
Rather than focusing on benchmarks alone, I discuss the trade-offs I encountered
in terms of performance, memory behavior, concurrency control, and engineering
complexity.

The goal of this post is to provide a practitioner-oriented perspective on how
different language and runtime choices affect the design of scalable graph
processing pipelines in real-world systems.

## Problem Context: Attack Chain Analysis
In security analytics, we often need to trace propagation paths through system dependency graphs. A typical scenario involves:
- Node: system entities (processes, files, network connections)
- Edges: dependencies or information flows
- Graph size: 10⁶ to 10⁸ nodes, 10⁷ to 10⁹ edges
- Query pattern: Breadth-first search (BFS) or reachability analysis from multiple seed nodes
- Constraint: real-time or near-real-time response requirements

The challenge is to parallelize graph traversal while managaing:
1. Load balancing across workers
2. Memory overhead from duplicate work
3. Synchronization costs
4. Cache efficiency

## Approach 1: Python with Numba JIT (just-in-time)

Implementation:

```

```