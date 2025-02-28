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
   - **Reading Notes**: [Readme](./notes/cxl-memory-expansion.md)

#### **2. Managing Memory Tiers with CXL in Virtualized Environments**
   - **Authors**: Yuhong Zhong, Daniel S. Berger, et al.
   - **Conference**: OSDI 2024
   - **Link**: [Paper Link](https://www.usenix.org/conference/osdi24/presentation/zhong-yuhong)
   - **Code Repository**: [Memstrata](https://bitbucket.org/yuhong_zhong/memstrata)
   - **Summary**: Introduces Memstrata, a multi-tenant memory allocator that optimizes performance by reducing inter-VM contention and memory tiering overhead in CXL-based systems.
   - **Reading Notes**: [Readme](./notes/2024/Memstrata-osdi24.md)

#### **3. An Introduction to Compute Express Link (CXL)**
   - **Authors**: Debendra Das Sharma, Robert Blankenship, et al.
   - **Source**: Intel Technical Paper
   - **Link**: [Paper Link](./CXL-intro.pdf)
   - **Summary**: Provides an overview of CXL, its evolution from CXL 1.0 to 3.0, and its impact on datacenter architectures.
   - **Reading Notes**: [Readme](./notes/cxl-intro.md)

### 2023
#### **1. TPP : Transparent Page Placement for CXL-Enabled Tiered-Memory**
   - **Authors**: Hasan Al Maruf, Hao Wang et al.
   - **Source**: ASPLOS 23
   - **Link**: [Paper Link](https://dl.acm.org/doi/10.1145/3582016.3582063)
   - **Summary**: Introduced an migration mechanism named TPP to transparent migrate pages between local memory and CXL-Memory w/o incur high overhead.
   - **Reading Notes**: [Readme](./notes/2023/TPP-ASPLOS.md)

---

### Persistent Memory (PMEM)

#### **1. An Empirical Guide to the Behavior and Use of Scalable Persistent Memory**
   - **Authors**: Jian Yang, Juno Kim, et al.
   - **Conference**: FAST 2020
   - **Link**: [Paper PDF](./PM-Study-FAST20.pdf)
   - **Summary**: Investigates Intel Optane Persistent Memory performance, best practices for software optimization, and re-evaluates prior PMEM research assumptions.
   - **Reading Notes**: [Readme](./notes/pm-study-fast20.md)

---

### TBC

