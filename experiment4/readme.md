# Experiment 4: Cache Coherence (MSI, MESI, MOESI) and NUMA Memory Access

## 1. Overview

This experiment explores how cache coherence protocols maintain consistent data across multiple CPU cores and how NUMA (Non-Uniform Memory Access) affects memory access performance.

The experiment focuses on:

- Studying MSI, MESI and MOESI cache coherence protocols.
- Understanding cache-line invalidation and false sharing.
- Comparing adjacent and padded memory layouts.
- Measuring local and remote NUMA memory access.
- Evaluating execution time, correctness and scalability with different thread counts.

## 2. Objectives

1. Understand why cache coherence is necessary in multi-core processors.
2. Simulate MSI, MESI and MOESI protocols and compare their bus transactions.
3. Observe the performance impact of false sharing.
4. Compare adjacent and padded counter layouts.
5. Study NUMA memory placement and its effect on execution time.
6. Analyze performance as the number of threads increases.

## 3. Technologies Used

| Technology | Purpose |
| --- | --- |
| Ubuntu 22.04 LTS | Operating system |
| VMware Workstation Pro | Virtualization |
| GCC Compiler | Compiling C programs |
| OpenMP | Parallel programming |
| Python 3 | Cache coherence simulation |
| numactl | NUMA CPU and memory binding |
| hwloc | Hardware topology information |

## 4. Theory and Concepts

### 4.1 Cache Coherence

In a multi-core processor, each CPU core may have its own private cache. When multiple cores access the same memory location, they may hold separate copies of the data.

If one core modifies its copy, other cores must be informed so they do not continue using stale data.

Cache coherence protocols ensure that all cores maintain a consistent view of shared memory.

### 4.2 MSI Protocol

MSI uses three states:

| State | Meaning |
| --- | --- |
| Modified (M) | Cache has the only valid, modified copy. |
| Shared (S) | Cache holds a clean copy that may be shared. |
| Invalid (I) | Cache line is not valid. |

A write to a shared cache line requires an upgrade transaction and invalidates other copies.

### 4.3 MESI Protocol

MESI extends MSI by introducing the **Exclusive (E)** state.

| State | Meaning |
| --- | --- |
| Modified (M) | Exclusive modified copy. |
| Exclusive (E) | Only copy, but clean. |
| Shared (S) | Clean copy shared with other caches. |
| Invalid (I) | Invalid cache line. |

A cache line in the Exclusive state can transition to Modified silently when the same core writes to it. This avoids an additional bus transaction.

### 4.4 MOESI Protocol

MOESI extends MESI by introducing the **Owned (O)** state.

The Owned state allows one cache to hold a modified line while supplying its data to other caches. This can avoid an immediate write-back to main memory.

The five states are:

- Modified (M)
- Owned (O)
- Exclusive (E)
- Shared (S)
- Invalid (I)

### 4.5 False Sharing

False sharing occurs when two threads modify different variables that happen to occupy the same cache line.

Even though the threads access different variables, cache coherence operates on entire cache lines. This causes unnecessary invalidations and cache-line transfers between cores.

**Solution:** Padding or alignment can place frequently modified variables on separate cache lines.

### 4.6 NUMA (Non-Uniform Memory Access)

In NUMA systems, memory is divided between different nodes.

- **Local memory:** Memory attached to the same NUMA node as the CPU.
- **Remote memory:** Memory attached to another NUMA node.

Local memory access is generally faster because remote access requires communication across an interconnect.

Linux commonly uses a first-touch policy, where memory pages are placed according to the node of the thread that first accesses them.

The `numactl` utility can control CPU and memory placement.

## 5. Programs Implemented

| No. | Program | Description |
| --- | --- | --- |
| 1 | `coherence_sim.py` | Simulates MSI, MESI and MOESI cache coherence protocols. |
| 2 | `false_sharing.c` | Compares adjacent and padded counter layouts. |
| 3 | `numa_sweep.c` | Measures NUMA memory access performance. |
| 4 | `compare_numa.c` | Compares execution time, correctness and scalability. |

## 6. Installation and Setup

Update Ubuntu packages:

```bash
sudo apt update
```

Install the required tools:

```bash
sudo apt install build-essential numactl hwloc python3 -y
```

Verify GCC:

```bash
gcc --version
```

Verify OpenMP support:

```bash
echo '#include <omp.h>' | gcc -E -fopenmp -xc - > /dev/null && echo "OpenMP OK"
```

Check CPU and NUMA configuration:

```bash
lscpu | grep -E "Socket|Core|Thread|NUMA"
numactl --hardware
lscpu -e
```

## 7. Compilation

Compile the C programs from the project directory.

### False Sharing

```bash
gcc -O2 false_sharing.c -fopenmp -o false_sharing
```

### NUMA Memory Sweep

```bash
gcc -O2 numa_sweep.c -fopenmp -o numa_sweep
```

### Combined Comparison

```bash
gcc -O2 compare_numa.c -fopenmp -o compare_numa
```

The Python simulator does not require compilation.

## 8. Execution

### Program 1: Cache Coherence Simulation

```bash
python3 coherence_sim.py 1000
```

This program simulates MSI, MESI and MOESI protocols using two cores and two counters.

It displays:

- Cache-state transitions.
- Bus transactions.
- Invalidations.
- Write-backs to memory.
- Cache-to-cache transfers.

It also compares adjacent and padded memory layouts.

### Program 2: False Sharing

Run with adjacent counters:

```bash
OMP_NUM_THREADS=2 ./false_sharing adjacent
```

Run with padded counters:

```bash
OMP_NUM_THREADS=2 ./false_sharing padded
```

This program compares the execution time of two memory layouts and displays the final counter value.

### Program 3: NUMA Memory Sweep

First, identify the available NUMA nodes:

```bash
numactl --hardware
```

Run with local memory placement:

```bash
OMP_NUM_THREADS=2 numactl --cpunodebind=0 --membind=0 ./numa_sweep
```

Run with remote memory placement:

```bash
OMP_NUM_THREADS=2 numactl --cpunodebind=0 --membind=1 ./numa_sweep
```

Replace node numbers according to the available NUMA configuration.

### Program 4: Combined Comparison

```bash
./compare_numa
```

This program performs two comparisons:

**Table 1: False Sharing**

- Adjacent and padded layouts.
- 1, 2 and 4 threads.
- Final counter value.
- Correctness verification.
- Execution time.

**Table 2: NUMA Memory Sweep**

- 512 MB memory buffer.
- 1 and 2 threads.
- Execution time for multiple passes.

To compare local and remote memory placement:

```bash
numactl --cpunodebind=0 --membind=0 ./compare_numa
```

```bash
numactl --cpunodebind=0 --membind=1 ./compare_numa
```

## 9. Observations and Results

### 9.1 Protocol Comparison

| Protocol | Main Observation |
| --- | --- |
| MSI | Requires an upgrade transaction when writing to a shared line. |
| MESI | Exclusive state avoids an upgrade transaction for private data. |
| MOESI | Owned state allows cache-to-cache transfers without an immediate memory write-back. |

The simulation demonstrates that all three protocols maintain consistency but differ in the amount and type of coherence traffic.

### 9.2 False Sharing Comparison

| Layout | Expected Behaviour |
| --- | --- |
| Adjacent | More cache-line bouncing and potentially higher execution time. |
| Padded | Less cache-line interference and generally better performance. |

Both layouts should produce the correct final counter value. Their execution times may differ because of cache coherence traffic.

### 9.3 NUMA Comparison

| Memory Placement | Expected Behaviour |
| --- | --- |
| Local | Generally lower memory access latency. |
| Remote | Generally higher memory access latency on multi-socket systems. |

On a single-socket computer or virtual machine, local and remote memory timings may be similar because virtual NUMA nodes may share the same physical memory.

### 9.4 Performance and Scalability

The comparison program evaluates how execution time changes when the number of threads increases.

The results help identify:

- The performance cost of false sharing.
- The benefits of padding.
- The effect of NUMA memory placement.
- The relationship between thread count and execution time.

**Note:** Actual execution times depend on the hardware, number of CPU cores, virtual machine configuration and system load.

## 10. Screenshots

<img width="998" height="630" alt="1794475002177_Screenshot_2026-11-12_144330" src="https://github.com/user-attachments/assets/5c2bf23d-c3ae-47df-8bd6-be49f9b86834" />
<img width="1012" height="627" alt="1794475002178_Screenshot_2026-11-12_144402" src="https://github.com/user-attachments/assets/07a12099-612c-4412-adf2-fe460e4d738a" />
<img width="1007" height="632" alt="1794475002179_Screenshot_2026-11-12_144421" src="https://github.com/user-attachments/assets/a41967af-6797-4e98-a064-71438f5d94ed" />
<img width="1001" height="630" alt="1794475015973_Screenshot_2026-11-12_144439" src="https://github.com/user-attachments/assets/d671dc5a-68fe-47d3-9fdf-536cac13c28c" />
<img width="1007" height="622" alt="1794475015974_Screenshot_2026-11-12_144457" src="https://github.com/user-attachments/assets/55af55e8-0de9-4082-9b52-83c76ed3f651" />
<img width="1008" height="630" alt="1794475015974_Screenshot_2026-11-12_144517" src="https://github.com/user-attachments/assets/90a442e7-0c15-4fa0-934a-5b69f98e50a2" />
<img width="1012" height="632" alt="1794475015975_Screenshot_2026-11-12_144537" src="https://github.com/user-attachments/assets/e608e77d-9ce3-446c-a8c4-2f5310eea1a9" />
<img width="1013" height="632" alt="1794475015976_Screenshot_2026-11-12_144552" src="https://github.com/user-attachments/assets/b59ada10-a058-45b2-baea-4ab32553c5cc" />

### 10.1 Cache Coherence Protocol Simulation

MSI, MESI and MOESI cache-state transitions and protocol statistics.

### 10.2 False Sharing Experiment

Execution comparison between adjacent and padded counter layouts.

### 10.3 NUMA Memory Sweep

Local and remote NUMA memory access measurements.

### 10.4 Combined Comparison Program

Execution-time, correctness and scalability comparison across different thread counts.

### 10.5 Additional NUMA Results

Additional terminal output for local and remote memory configurations, if captured separately.

> **Note:** Place your actual screenshots inside a folder named `screenshots` in the repository. Update the image filenames above to match your saved files. Remove any screenshot section for which you do not have an image.

## 11. Conclusion

This experiment demonstrates how cache coherence protocols maintain consistent data across multiple CPU cores and how their states affect memory transactions.

The MSI, MESI and MOESI simulation highlights the differences in coherence traffic. The false-sharing experiment demonstrates how placing independent variables on the same cache line can reduce parallel performance, while padding helps minimize this overhead.

The NUMA experiment shows that memory placement can influence access time, particularly on multi-socket systems.

Overall, the experiment highlights the importance of cache coherence, efficient memory layout and NUMA-aware memory placement in designing scalable parallel programs.

---

**Topics:** Cache Coherence \| MSI \| MESI \| MOESI \| False Sharing \| OpenMP \| NUMA \| Parallel Computing
