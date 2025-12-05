# SHADE: Enable Fundamental Cacheability for Distributed Deep Learning Training
Fast '23 – Summary

## Overview
Distributed deep learning (DLT) workloads issue many small, random, read-only I/O operations, making them difficult to cache. Traditional caching (LRU/LFU) fails due to weak locality and the fact that each sample is accessed only once per epoch.

SHADE introduces **importance-driven sampling and caching** that convert DLT workloads into cacheable workloads.

## Key Ideas

### 1. Fine-Grained Per-Sample Importance
SHADE decomposes minibatch loss into **per-sample entropy loss**.  
This provides a fine-grained signal about which samples contribute most to training.

### 2. Rank-Based Importance Scores
Raw loss values across minibatches are not comparable.  
SHADE assigns **rank-based scores** within each batch and applies log-scaling.

Benefits:
- Importance becomes comparable across batches and epochs.
- Top X% of each batch correlates with global top X% of the dataset.

### 3. PADS: Priority-Based Adaptive Data Sampling
SHADE builds a **multinomial distribution** from importance scores and repeats high-importance samples in the next epoch.

This creates:
- Predictable, importance-skewed access patterns  
- Implicit locality that caching systems can exploit  
- Faster model convergence

## Architecture

SHADE consists of a **Control Layer** and **Data Layer**.

### Control Layer
- Computes per-sample importance.
- Produces rank-based scores.
- Generates an importance-weighted sample order for the next epoch (PADS).
- Sends only sample indices to the data layer.

### Data Layer
Implements importance-aware caching using:

#### Priority Queue (PQ)
Tracks all currently cached samples, sorted by importance.  
`PQ.min` = least-important cached sample.

#### Ghost Cache
Tracks all previously accessed samples:  
`sample_id → {importance_score, access_frequency}`  
Stores metadata only (no sample data).

Used to decide whether an evicted sample should return to the cache.

## APP Caching Policy (Adaptive Priority-Aware Prediction)

<figure>
       <img src="../../imgs/SHADE-FAST23/Workflow.png" alt="Figure 3" style="width:70%; height:auto;">
       <figcaption>Figure 1: Architecture of SHADE.</figcaption>
   </figure>


This yields **implicit prefetching**:
important samples are cached early because PADS will access them repeatedly.

## Results

- **4.5×** higher read hit ratio
- **2.7×** faster throughput
- **3×** faster model convergence
- Can **outperform Belady’s MIN**, because SHADE shapes future access patterns instead of only predicting them.

## Takeaway
SHADE makes deep learning training **cacheable** by combining:
- Fine-grained importance scoring  
- Rank-based comparability  
- Importance-driven sampling (PADS)  
- Importance-aware caching (APP)

The system creates locality instead of relying on it.

