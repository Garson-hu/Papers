# PolyStore: Exploiting Combined Capabilities of Heterogeneous Storage

FAST '25, Rutgers University + EPFL + Samsung Semiconductor (Yujie Ren et al., Feb 25-27 2025, Santa Clara)

**In one sentence**: Storage hierarchies waste bandwidth because they always put the fast device *on top*; PolyStore lays heterogeneous storage devices out **horizontally** as a meta-layer above per-device-optimized file systems (NOVA on PM, ext4/F2FS on flash), so a single logical file is striped across all of them at 2 MB granularity. Result: **up to 9.38x** over state-of-the-art caching/tiering on microbenchmarks and **1.52x-2.02x** on real applications.

# Background

1. Storage media stopped being a clean ladder. PM (Optane) is byte-addressable with low latency; NVMe SSDs have high throughput; CXL-based SSDs offer low latency *and* high bandwidth; SATA SSDs give capacity. The paper calls this the **"non-hierarchical" trend** in **HSDs** (heterogeneous storage devices).

2. Prior art all assumes hierarchy, in three flavors:
   - **Caching** (bcache, Orthus, P2CACHE): fast device absorbs writes / accelerates reads, slow device backs it.
   - **Tiering** (Strata, Ziggurat, SPFS): data lives *exclusively* on one tier, migrated by hotness policy.
   - **Application-directed** (NoveLSM, DBMS WAL placement): app-level knowledge decides placement.

3. **Table 1 — what nobody supports before PolyStore.**

| HSD property | Caching | Tiering | App-specific | **PolyStore** |
|---|---|---|---|---|
| Cumulative **read** bandwidth utilization | partial (Orthus only) | ✗ | ✗ | **✓** |
| Cumulative **write** bandwidth utilization | ✗ | ✗ | ✗ | **✓** |
| Heterogeneity-aware DRAM caching | partial (P2CACHE only) | ✗ | ✗ | **✓** |
| Cross-device durability & crash consistency | ✗ | ✓ | ✓ | **✓** |

   The interesting cell is **cumulative write**: *no* prior system gets it. Orthus reaches cumulative *reads*, but only after the fast device is already saturated, and its block-layer implementation cannot guarantee atomicity or durability, so it cannot be extended to writes.

# Motivation: three ways hierarchy loses

1. **Fast-on-top forfeits the combined bandwidth.** Even Orthus, the best non-hierarchical caching design, still performs *writes* hierarchically.
2. **Prioritizing the fast device creates contention on it.** PM in particular degrades past ~8 threads. Eager placement on fast storage also burns bandwidth on eviction and migration.
3. **DRAM caching is static.** The Linux page cache and DRAM+PM designs (P2CACHE, Ziggurat) do not adapt to the varying performance gaps between devices in an HSD set.

<figure>
       <img src="../../imgs/PolyStore-FAST25/M1.png" alt="Figure 1" style="width:70%; height:auto;">
       <figcaption>Figure 1: Inefficient hardware utilization. 32 threads, direct I/O without DRAM cache, each thread on its own 2 GB file (Config I). The red line is the combined PM+NVMe bandwidth — every existing system sits far below it.</figcaption>
   </figure>

The read case is instructive per system: Orthus must *admit* data NVMe -> PM on a miss, so random reads turn into concurrent PM **writes**; Strata has the same problem; SPFS refuses to promote NVMe data until a *write* touches it, so with the page cache off its reads are stuck at NVMe speed.

# Design of PolyStore

<figure>
       <img src="../../imgs/PolyStore-FAST25/D1.png" alt="Figure 2" style="width:85%; height:auto;">
       <figcaption>Figure 2: PolyStore high-level design. A user-level runtime (Poly-index, Poly-cache, Poly-placement, Poly-persist) intercepting POSIX I/O, plus a thin VFS kernel component (PolyOS) for sharing/fairness/security, sitting atop unmodified device-optimized file systems.</figcaption>
   </figure>

**The central architectural bet**: don't write a new multi-device file system — write a *meta layer* and reuse mature, hardware-optimized file systems underneath (NOVA for PM, ext4/F2FS for flash). Split it across user and kernel: the **runtime** maps data across devices, the **OS component** keeps protection, sharing and fairness.

## Structures (§4.3)

- **Poly-inode** — one logical file → N physical files, one per device. Holds per-device fds and refcounts.
- **Poly-index** — a **range-tree** (augmented red-black tree keyed by non-overlapping `(low, high)` intervals). Each node is the unit of locking and covers **2 MB by default**, chosen because (1) it balances memory footprint against concurrency and (2) it matches **Linux's maximum block I/O size for a single request**, so a whole range evicts in one request. Per-node reader-writer lock. **128 B in memory / 96 B on disk**, so a 1 TB file costs only **64 MB (0.0064%)**.
- **Hybrid namespace** — directory hierarchy on the fast device (`/fast/d1/f1`), **flat hashed names** on slow devices (`/slow/hash("polyfs/d1/f1")`), to avoid paying random-access name resolution on the slow tier.

Both Poly-inode and Poly-index live in memory-mapped files on the faster storage.

## Heterogeneity-aware file system ops (§4.4)

- **Adaptive file creation.** Start every file on *one* device. Only when it grows past the 2 MB threshold and bandwidth saturates does PolyStore create additional physical files on other devices. Small files never pay the cost of two file systems.
- **Write indexing** handles three cases: unindexed range (create nodes on demand), partially indexed (I/O the whole range, backfill nodes), fully indexed (in-place update of modified blocks only, to limit write amplification).
- **Static thread→device mapping** as the baseline: offline, using `fio`, determine how many threads saturate each device; fill the fastest device first, spill the rest to slower devices. **A single write's blocks always land on a single device** — that is what makes ordering and crash consistency tractable.
- **POSIX appends** are preserved two ways: all physical files are opened in append mode (serialization within a device), and Poly-index **timestamps** establish global order across devices. The timestamps also constrain eviction: when `blkN` is evicted, all of `blk0..blkN-1` must be too. On failure, recovery truncates everything after the first unrecoverable block.

## Poly-placement: dynamic thread remapping (§4.5.1)

Finding the optimal thread→device mapping is knapsack-hard, so PolyStore uses a **greedy three-state machine** per adjacent device pair, re-evaluated every **200 ms epoch**, with a configurable **50%** bandwidth-change threshold:

<figure>
       <img src="../../imgs/PolyStore-FAST25/D2.png" alt="Algorithm 1" style="width:75%; height:auto;">
       <figcaption>Algorithm 1: Thread mapping and remapping. States are low_to_high (initial), high_to_low, and STABLE. Note lines 29-30: a global bandwidth drop resets the whole mapping.</figcaption>
   </figure>

- `low_to_high` (initial): keep moving threads onto the faster device while its bandwidth improves; if it *drops*, flip to `high_to_low`.
- `high_to_low`: shed threads to slower storage to relieve contention.
- `STABLE`: stop remapping. Only a **global** bandwidth drop (line 29) forces a reset to the initial mapping.

This is the direct answer to the contention problem in §3 — instead of assuming PM is best, it *measures* whether adding another thread to PM still helps.

## Migration (§4.5.2)

Two 2-bit-state migrations over Poly-index (`MORE_FREQUENT` / `LESS_FREQUENT`, threshold 50% of accesses by default):

- **`BW_Move`** (bandwidth-aware): promote hot blocks from slow to fast storage — **but only if the target's read *and* write bandwidth are not saturated** and it has free space. This is precisely what Orthus and SPFS get wrong.
- **`CAP_Move`** (capacity-aware): when a device crosses a **90% watermark**, a background thread demotes blocks downward.

Mechanism is append → update Poly-index → truncate source + mark garbage, all **wrapped in a transaction**. Garbage is collected by `fallocate` hole punching or by compaction.

## Poly-cache: heterogeneity-aware DRAM caching (§4.6)

User-level, **bypassing the OS page cache** entirely — no double caching, no kernel trap on a hit. Buffers hang off Poly-index nodes (2 MB, backed by huge pages where available). Eviction is a **two-level LRU** (file-level LRU for inactive files, per-file range LRU inside), budgeted per application with **cgroups**, executed by a thread pool that starts at one thread per storage type and **scales up with observed bandwidth** — evictions themselves exploit cumulative bandwidth. Because the layout is horizontal, eviction can also *redirect* a block to a different device than the one it came from, which quietly doubles as a placement optimization.

Two device-specific policies:
- **Bypass the cache for PM reads.** PM reads are already fast; a DRAM copy adds overhead with no gain. Blocks are read directly from a PM mapping until a write touches the range. Writes still get buffered, because PM writes are the slow direction.
- **Dynamic cache ratio for SSDs.** Give the *slower* device (SATA vs NVMe) a larger share of the cache, adjusted from measured bandwidth/latency and the access ratio between devices.

## Poly-persist + PolyOS (§4.7-4.8)

- **Atomicity** for `rename`/`link`/`truncate`: a journal with a global log of uncommitted files, per-file metadata, and an operational log. Commit = serially commit to the device file system, then update Poly-inode/Poly-index. Unrecoverable files are marked inaccessible and left to `fsck`.
- **Crash consistency**: Poly-inode log entries are sized to **one cache line**, Poly-index range nodes to **two**, committed with `clflush` + fences on PM; without PM, memory-mapped files plus `msync`. Recovery waits for the underlying file systems to mount first, then replays.
- **Common minimal durability**: since NOVA gives full data+metadata durability while ext4 ships with data journaling off, PolyStore levels down to a **uniform baseline** (metadata-only in the NOVA+ext4 config) rather than pretending the guarantees are uniform. Honest, and a real limitation.
- **Security**: a user-level cache is a corruption risk — a read-only process could append a bogus cache buffer into the shared Poly-index. PolyOS handles this by **revoking write access to the memory-mapped Poly-index file** when a file is shared, forcing all subsequent index updates through the kernel. Fairness across applications uses Linux I/O throttling.

# Evaluation

| | Config I | Config II | Config III |
|---|---|---|---|
| Faster | 256 GB Optane PM | 1 TB NVMe SSD | 256 GB Optane PM |
| Slower | 1 TB NVMe SSD | 2 TB SATA SSD | 1 TB NVMe SSD |
| Slowest | - | - | 2 TB SATA SSD |

Bandwidth (write / read): PM **4.6 / 13.2 GB/s**, NVMe **1.2 / 1.2 GB/s**, SATA **560 / 530 MB/s**. NOVA on PM, ext4 on flash. Baselines: PM-only, NVMe-only, SATA-only, **Orthus** (caching), **Strata** / **SPFS** (tiering), **P2CACHE** (PM+DRAM). *No baseline supports more than two device types*, so Config III is PolyStore-only.

<figure>
       <img src="../../imgs/PolyStore-FAST25/E1.png" alt="Figure 3" style="width:100%; height:auto;">
       <figcaption>Figure 3: Microbenchmark, direct I/O without DRAM cache, Config I. Red dotted lines = maximum combined PM+NVMe bandwidth. This is the paper in one figure: every prior system sits far under the line, PolyStore-dynamic nearly touches it.</figcaption>
   </figure>

## Results at a glance

| Question | Answer |
|---|---|
| Cumulative bandwidth (seq write, Config I) | **92.3% of combined** at 32 threads; 1.48x over PolyStore-static, **up to 9.38x** over caching/tiering |
| Latency, 32 threads | write **23.5 µs** vs Orthus 150.4 (**6.38x**); read 22.9 vs 54.3. Hierarchical systems spike because everything funnels into PM |
| Sequential read | 1.11x over PM-only — modest, but PM-only *is* the read king at 13.2 GB/s, so any gain means NVMe reads are genuinely additive |
| Works without PM? (NVMe+SATA) | **1.87x** over NVMe-only (write), 2.23x over Orthus (read). Not an Optane artifact |
| Shared 64 GB file, 16 readers + 16 writers | **2.95x / 3.04x** over NVMe-only — range-level locks dissolve the per-inode `rw-lock` |
| Three devices | **91.7%** of combined bandwidth. Horizontal scaling scales |
| Does the underlying FS matter? | NOVA/F2FS beats ext4-DAX/ext4 by **1.63x** — validates the whole "reuse mature per-device FSes" premise |
| Poly-cache | **6.31x** when the working set fits; under 16:1 pressure **3.18x** over PM-only, because eviction itself uses cumulative bandwidth |
| Metadata-heavy (Filebench) | **3.12x** over PM-only on Fileserver; gains over the *other* systems are marginal and come from skipping syscalls, not bandwidth |
| Multi-app / crash recovery | 1.96x over P2CACHE without hurting Redis; **2.91x** post-recovery throughput (recovery itself is slightly *slower* — physical files restore before Poly-inodes) |
| GraphWalker MS-PPR | **2.46x fewer PM writes** than Orthus, graph load 1.84x / 1.62x faster than Orthus / SPFS — because it checks PM write saturation before promoting |

<figure>
       <img src="../../imgs/PolyStore-FAST25/E7.png" alt="Figure 9" style="width:100%; height:auto;">
       <figcaption>Figure 9: RocksDB + Redis, Config I. YCSB gains are 1.52x (A) and 2.02x (F); F wins biggest because Poly-cache evicts NVMe-resident blocks *into PM*, speeding up reaccess.</figcaption>
   </figure>

# Takeaways / My Notes

1. **The thesis is about the *direction of prioritization*, not a new device.** Every baseline decides *a priori* that the fast device wins. PolyStore replaces that prior with a closed loop — Algorithm 1's state machine and `BW_Move`'s saturation check. One number to keep: **92.3% of combined bandwidth**.

2. **"Meta-layer over mature file systems" is the reusability argument, and the 1.63x NOVA/F2FS result is what backs it.** Strata had to *write* a cross-media file system; PolyStore inherits NOVA's per-CPU scalability and F2FS's flash-friendly logging for free. Should extend to a CXL-SSD plus its file system unchanged.

3. **2 MB is load-bearing.** It is simultaneously the Poly-index lock granularity, the Poly-cache buffer, the huge-page size, the small-vs-large file threshold, and **Linux's max single block I/O request** — that last alignment is why a whole range evicts in one request.

4. **Eviction as placement is the sleeper idea.** Horizontal layout lets Poly-cache evict a block to *any* device, not back where it came from. That produces the best YCSB-F result, and no hierarchical cache can do it by construction.

5. **Three answers to one question.** [[TPP-ASPLOS]] and [[Memstrata-OSDI24]] *manage* a fast/slow gap; [[Beluga-SIGMOD26]] *closes* it in hardware; PolyStore *denies* the hierarchy exists. The deciding variable is the size of the gap — PolyStore's PM/NVMe write gap (~4x) is small enough for striping to win, which would not hold for DRAM vs. SATA.

6. **Where I'm skeptical.** 9.38x is a microbenchmark best case — applications land at 1.52x-2.02x. **Common minimal durability** means one weak file system silently downgrades guarantees for the whole logical file. Static mapping is bootstrapped from offline `fio` profiling, a deployment cost the paper doesn't price. And with Optane discontinued, the number that actually matters for the future is the weaker Config II result (**1.87x**).

## Acknowledgement
All figures are cropped from the authors' paper (FAST '25 proceedings, pp. 539-555).

## Links
- Paper: [USENIX FAST '25](https://www.usenix.org/conference/fast25/presentation/ren), [PDF](https://www.usenix.org/system/files/fast25-ren.pdf)
- Code: [RutgersCSSystems/PolyStore](https://github.com/RutgersCSSystems/PolyStore) (artifact evaluated: available / functional / reproduced; Linux 5.1.0 + NOVA, Ubuntu 20.04.5)
- Local PDF: `/Users/garson/Research/NCSU/Paper/PolyStore_FAST25.pdf`
