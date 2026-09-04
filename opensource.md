---
title: Open Source
permalink: /opensource/
---

## Featured Projects

### Trace2PoC

**Runtime-evidence-guided agentic vulnerability discovery for MCP servers and AI agent tools.**

Trace2PoC turns source-level vulnerability hypotheses into minimized, replayable proofs of concept. It analyzes MCP server and agent-tool repositories, plans adversarial tool interactions, executes them in isolation, and validates impact using deterministic runtime evidence. The language model proposes what to test; runtime oracles decide whether a vulnerability is real.

**Status:** Design and early implementation; no public repository yet
**Stack:** Agentic workflows, MCP, sandboxed execution, runtime tracing
**Focus:** Vulnerability discovery, runtime verification, PoC minimization

---

### Agentic Security Lab

**Security workbench for attacking, observing, and securing agentic AI and MCP systems.**

Agentic Security Lab brings together automated attack simulation, sandboxed tool execution, runtime monitoring, and detection testing. It provides vulnerable agent and MCP test targets, attack runners, detectors, binary-analysis components, and reproducible reporting workflows for agentic security research.

**Stack:** Python, Docker, MCP, sandboxed execution, runtime monitoring
**Focus:** Agentic AI security, attack simulation, detection validation
**Links:** [GitHub](https://github.com/Wapiti08/agentic-security-lab)

---

### MCP-SandboxScan

**WASM-based secure execution and hybrid analysis framework for MCP tools.**

MCP-SandboxScan analyzes the behavior of LLM agent tool integrations in a sandboxed runtime. It is designed to detect risky file access, network behavior, suspicious command execution, secret exposure, and unsafe tool-server patterns.

**Stack:** Rust, WASM, Python, MCP, runtime tracing
**Focus:** Agentic AI security, tool sandboxing, runtime behavior analysis
**Links:** [GitHub](https://github.com/Wapiti08/MCP-SandboxScan) · [Paper](https://arxiv.org/pdf/2601.01241)
---

### SynthChain

**APT-style software supply chain attack simulation framework.**

SynthChain simulates realistic software and AI supply chain compromise scenarios across package ecosystems, CI/CD workflows, Docker/ML pipelines, and cloud-connected development environments.

**Stack:** Python, Go, NPM, Docker, CI/CD, provenance graphs
**Focus:** Supply chain attack simulation, runtime observability, detection benchmark design
**Links:** [zenodo](https://zenodo.org/records/22133892) · [Paper / Preprint](https://arxiv.org/pdf/2603.16694)
