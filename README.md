# Paper Reading Notes

This repository contains structured notes on research papers I have read, categorized by year and topic. Each paper includes a brief description, relevant links, and my summarized reading notes.

## Table of Contents

- [2024](#2024)
  - [CXL-Related Papers](#cxl-related-papers-2024)
  - [Persistent Memory (PMEM)](#persistent-memory-2024)
  - [Operating Systems and Virtualization](#operating-systems-and-virtualization-2024)

---

## CXL-Related Papers

### 2024

#### **1. CXL Memory Expansion: A Closer Look on Actual Platform**
   - **Authors**: Vinicius Petrucci, Eishan Mirakhur, et al.
   - **Source**: White Paper
   - **Link**: [Paper Link](./cxl-memory-expansion-a-close-look-on-actual-platform.pdf)
   - **Code Repository**: 
   - **Summary**: Discusses real-world performance and implications of CXL memory expansion, including bandwidth, capacity scaling, and NUMA-tiering strategies.
   - **Reading Notes**: [Reading notes](./notes/cxl-memory-expansion.md)

#### **2. Managing Memory Tiers with CXL in Virtualized Environments**
   - **Authors**: Yuhong Zhong, Daniel S. Berger, et al.
   - **Conference**: OSDI 2024
   - **Link**: [Paper Link](https://www.usenix.org/conference/osdi24/presentation/zhong-yuhong)
   - **Code Repository**: [Memstrata](https://bitbucket.org/yuhong_zhong/memstrata)
   - **Summary**: Introduces Memstrata, a multi-tenant memory allocator that optimizes performance by reducing inter-VM contention and memory tiering overhead in CXL-based systems.
   - **Reading Notes**: [Reading notes](./notes/2024/Memstrata-osdi24.md)

#### **3. An Introduction to Compute Express Link (CXL)**
   - **Authors**: Debendra Das Sharma, Robert Blankenship, et al.
   - **Source**: Intel Technical Paper
   - **Link**: [Paper Link](./CXL-intro.pdf)
   - **Summary**: Provides an overview of CXL, its evolution from CXL 1.0 to 3.0, and its impact on datacenter architectures.
   - **Reading Notes**: [Reading notes](./notes/cxl-intro.md)

### 2023
#### **1. TPP : Transparent Page Placement for CXL-Enabled Tiered-Memory**
   - **Authors**: Hasan Al Maruf, Hao Wang et al.
   - **Source**: ASPLOS 23
   - **Link**: [Paper Link](https://dl.acm.org/doi/10.1145/3582016.3582063)
   - **Summary**: Introduced an migration mechanism named TPP to transparent migrate pages between local memory and CXL-Memory w/o incur high overhead.
   - **Reading Notes**: [Reading notes](./notes/2023/TPP-ASPLOS.md)

#### **1. Pond: CXL-Based Memory Pohttps://github.com/MoatLab/Pondoling Systems for Cloud Platforms**
   - **Authors**: Huaicheng Li (Virginia Tech) et al
   - **Source**: ASPLOS 2023
   - **Link**: [Paper Link](https://dl.acm.org/doi/10.1145/3575693.3578835)
   - **Code**: [Github](https://github.com/MoatLab/Pond)
   - **Summary**: 
   - **Reading Notes**: [Reading notes](./notes/2023/Pond-ASPLOS.md)

---

## Persistent Memory (PMEM)

#### **1. An Empirical Guide to the Behavior and Use of Scalable Persistent Memory**
   - **Authors**: Jian Yang, Juno Kim, et al.
   - **Conference**: FAST 2020
   - **Link**: [Paper PDF](./PM-Study-FAST20.pdf)
   - **Summary**: Investigates Intel Optane Persistent Memory performance, best practices for software optimization, and re-evaluates prior PMEM research assumptions.
   - **Reading Notes**: [Reading notes](./notes/pm-study-fast20.md)

---

## Memory for LLM
#### **1. Efficient Memory Management for Large Language Model Serving with PagedAttention**
   - **Authors**: Woosuk Kwon, Zhuohan Li, et al.
   - **Conference**: SOSP 2023
   - **Link**: [Paper Link](https://arxiv.org/abs/2309.06180)
   - **Code Repository**: [vLLM](https://github.com/vllm-project/vllm)
   - **Summary**: vLLM introduced PagedAttention: it applies OS-style paging to the KV cache for LLM inference—splitting it into fixed-size blocks and mapping them with a page table to non-contiguous GPU memory—to reduce fragmentation, enable sharing/reuse, and manage memory more efficiently.
   - **Reading Notes**: [Reading notes](./notes/2023/vLLM-SOSP.md)


## Deep Learning Systems & Storage Optimization

#### **1. SHADE: Enable Fundamental Cacheability for Distributed Deep Learning Training**
   - **Authors**: Redwan Ibne Seraj Khan, Ahmad Hossein Yazdani, Ali R. Butt et al.
   - **Conference**: FAST 2023
   - **Link**: [Paper Link](https://www.usenix.org/conference/fast23/presentation/khan)
   - **Summary**: 
     SHADE introduces an importance-driven approach to make deep-learning training workloads cacheable. 
     It computes fine-grained per-sample importance, converts them into rank-based scores, and uses PADS (Priority-based Adaptive Data Sampling) 
     to shape future access patterns. The data layer uses an APP caching policy combining a priority queue and ghost cache to prefetch and retain important samples. This yields up to 4.5× higher cache hit ratio, 2.7× throughput improvement,  and faster model convergence.
   - **Reading Notes**: [Reading notes](./notes/2023/SHADE-FAST23.md)