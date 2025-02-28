# Background

1. The memory today is **tightly coupled** to CPU. 

- Memory is homogeneous, which mean they have the same type, latency, capacity and bandwidth etc.
- The rack-level memory power and cost increases with new hardware generations as shown in Figure 1.

<figure>
       <img src="../../imgs/TPP-ASPLOS23/B1.png" alt="Figure 1" style="width:50%; height:auto;">
       <figcaption>Figure 1: Trends in rack-level memory power and cose increases with new hardware generations over time.</figcaption>
   </figure>


2. CXL-based Heterogeneous Memory
The category of CXL memory can be categorized as shown in Figure 2.

<figure>
       <img src="../../imgs/TPP-ASPLOS23/B2.png" alt="Figure 2" style="width:60%; height:auto;">
       <figcaption>Figure 2: CXL memory abstraction and categories.</figcaption>
   </figure>


- The CXL memory have several strengths in flexible CPU and memory bus, which includes:
    - different memory capacity to bandwidth ratio
    - combine different generation of DIMMs
    - use cheaper and low power memory alternatives
    - utilize near memory accelerators


3. CXL-Memory Characteristics
- Byte addressable in same physical address space
    - transparent allocation with cache-line granular access
