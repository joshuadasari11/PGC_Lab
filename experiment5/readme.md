# Experiment 5: Neural Network Layers in CUDA

A CUDA C++ implementation of a small fully connected neural network forward pass (matrix multiplication + bias, ReLU, Sigmoid), benchmarked against an equivalent CPU implementation on an NVIDIA RTX 4500 Ada Generation GPU.

---

## Table of Contents

1. [Overview](#overview)
2. [System Configuration](#system-configuration)
3. [Environment Setup and Verification](#environment-setup-and-verification)
4. [Project Structure](#project-structure)
5. [Build and Run](#build-and-run)
6. [Step 1: CUDA Sanity Test](#step-1-cuda-sanity-test)
7. [Step 2: Neural Network Layers](#step-2-neural-network-layers)
8. [Performance Results](#performance-results)
9. [Analysis](#analysis)
10. [Troubleshooting](#troubleshooting)
11. [Conclusion](#conclusion)

---

## Overview

This experiment covers:

- Verifying the CUDA toolchain (driver, `nvcc`, Visual Studio build tools).
- Compiling and running a basic CUDA test program.
- Implementing neural network building blocks as CUDA kernels:
  - Matrix multiplication with bias
  - ReLU activation
  - Sigmoid activation
- Validating GPU output against a CPU reference.
- Comparing CPU and GPU execution time.

---

## System Configuration

| Component | Details |
|---|---|
| GPU | NVIDIA RTX 4500 Ada Generation |
| GPU Memory | 24570 MiB |
| Compute Capability | 8.9 (`sm_89`) |
| Max Power | 210 W |
| Driver Version | 610.47 |
| Driver Model | WDDM |
| CUDA Version (driver, `nvidia-smi`) | 13.3 |
| CUDA Toolkit (`nvcc`) | 13.4 (V13.4.92) |
| Host Compiler | Visual Studio 2026 Build Tools (Developer Command Prompt v18.12.0) |
| Architecture | `amd64` (host and target) |
| Shell | PowerShell / Command Prompt (Windows) |

---

## Environment Setup and Verification

### 1. Check the GPU and driver

```powershell
nvidia-smi
```

Key observations from the output:

- GPU: **NVIDIA RTX 4500 Ada Generation**
- Driver version: **610.47**
- CUDA UMD version: **13.3**
- Idle state: 38 C, P5 performance state, 14 W / 210 W, 1081 MiB / 24570 MiB used, 6% utilization
- Only desktop and browser processes (Explorer, Chrome, Edge, etc.) were using the GPU at the time, so the benchmark had the GPU essentially to itself.

**Screenshot:**

<img width="830" height="732" alt="Screenshot 2026-11-25 200812" src="https://github.com/user-attachments/assets/f791d9a6-d793-4112-b3b4-14df58af4239" />

### 2. Check the CUDA compiler

```powershell
nvcc --version
```

```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2026 NVIDIA Corporation
Built on Sun_Sep_01_02:23:07_Pacific_Standard_Time_2026
Cuda compilation tools, release 13.4, V13.4.92
Build cuda_13.4.r13.4/compiler.38855100_0
```

**Screenshot:**

<img width="572" height="115" alt="Screenshot 2026-11-25 200825" src="https://github.com/user-attachments/assets/a8919b72-f585-4d2a-a636-fc7ba63021cc" />

### 3. Confirm the `nvcc` location

```powershell
where.exe nvcc
```

```
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v13.4\bin\nvcc.exe
```

**Screenshot:**

<img width="638" height="35" alt="Screenshot 2026-11-25 200842" src="https://github.com/user-attachments/assets/332576a9-4a98-4e1d-928c-b16a0477726f" />

### 4. Open the Visual Studio Developer Command Prompt

`nvcc` on Windows needs the MSVC host compiler (`cl.exe`) on the path. First confirm the environment script exists, then launch it:

```powershell
Test-Path "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\Common7\Tools\VsDevCmd.bat"
```

```
True
```

```powershell
cmd.exe /k '"C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\Common7\Tools\VsDevCmd.bat" -arch=amd64 -host_arch=amd64'
```

```
**********************************************************************
** Visual Studio 2026 Developer Command Prompt v18.12.0
** Copyright (c) 2026 Microsoft Corporation
**********************************************************************
```

**Screenshot:**

<img width="1425" height="132" alt="Screenshot 2026-11-25 200902" src="https://github.com/user-attachments/assets/83b6e509-9b3f-4804-8afe-f8d02195754e" />

---

## Project Structure

```
Experiment5_CUDA/
|-- cuda_test.cu          # Basic CUDA sanity test
|-- cuda_test.exe         # Compiled sanity test
|-- neural_network.cu     # Neural network layers + CPU/GPU benchmark
|-- neural_network.exe    # Compiled benchmark
`-- README.md
```

Location used in this experiment:

```
C:\Users\PowerX\Documents\Experiment5_CUDA
```

---

## Build and Run

All commands are run from the **Developer Command Prompt** (see step 4 above).

### Go to the project folder

```bat
cd /d "%USERPROFILE%\Documents\Experiment5_CUDA"
```

### Build

```bat
nvcc -O2 -arch=sm_89 cuda_test.cu -o cuda_test.exe
nvcc -O2 -arch=sm_89 neural_network.cu -o neural_network.exe
```

| Flag | Meaning |
|---|---|
| `-O2` | Optimization level 2 |
| `-arch=sm_89` | Target the Ada Lovelace architecture (compute capability 8.9) |
| `-o <name>` | Output executable name |

### Run

```bat
cuda_test.exe
neural_network.exe
```

---

## Step 1: CUDA Sanity Test

A first test program (`cuda_test.cu`) was compiled without an explicit architecture flag:

```bat
nvcc -O2 cuda_test.cu -o cuda_test.exe
cuda_test.exe
```

```
Number of CUDA devices: 1
CUDA test completed successfully!
```

**Screenshot:**

<img width="771" height="192" alt="Screenshot 2026-11-25 201006" src="https://github.com/user-attachments/assets/0fed5e1e-f17c-48c4-a2a0-0542269bc4f6" />

It was then rebuilt for the exact GPU architecture and extended to print more detail:

```bat
nvcc -O2 -arch=sm_89 cuda_test.cu -o cuda_test.exe
cuda_test.exe
```

```
Number of CUDA devices: 1
GPU Name: NVIDIA RTX 4500 Ada Generation
GPU memory allocated successfully.
Kernel launched successfully.
Kernel execution completed.
GPU Results: 100 200 300
```

**Screenshot:**

<img width="850" height="193" alt="Screenshot 2026-11-25 201025" src="https://github.com/user-attachments/assets/21e421b4-17c7-4d98-893c-0c13ebfc12b6" />

This confirms that:

- The CUDA runtime detects the GPU.
- Device memory allocation works.
- Kernels launch and complete.
- Results are copied back to the host correctly.

---

## Step 2: Neural Network Layers

`neural_network.cu` implements a two-layer feed-forward network forward pass and compares it with a CPU reference.

### Network dimensions

| Layer | Shape |
|---|---|
| Input | 256 x 128 |
| Hidden | 256 x 64 |
| Output | 256 x 10 |

That is a batch of 256 samples, 128 input features, 64 hidden units, and 10 output units.

### CUDA kernels implemented

| Kernel | Operation |
|---|---|
| Matrix multiplication + bias | `Y = X * W + b` |
| ReLU | `y = max(0, x)` |
| Sigmoid | `y = 1 / (1 + e^(-x))` |

### Forward pass

```
Input (256x128)
   -> Matrix multiply + bias -> ReLU      -> Hidden (256x64)
   -> Matrix multiply + bias -> Sigmoid   -> Output (256x10)
```

### Kernel verification

```
Testing CUDA kernels...
Matrix multiplication + bias: PASSED
ReLU: PASSED
Sigmoid: PASSED

All kernel tests passed!
```

---

## Performance Results

The program was run twice. Both runs produced matching outputs.

### Run 1

```
========================================
 EXPERIMENT 5: NEURAL NETWORK LAYERS IN CUDA
========================================
GPU detected: NVIDIA RTX 4500 Ada Generation

Average CPU execution time: 0.717745 ms
Average GPU kernel-only time: 0.021043 ms
Average GPU end-to-end time: 0.091440 ms
Kernel-only speedup: 34.108168x
End-to-end speedup: 7.849355x

Maximum absolute output difference: 0.000000
CPU and GPU outputs: MATCH (within tolerance)

First 10 GPU output values:
0.498196 0.503101 0.508006 0.512911 0.517814 0.497729 0.502638 0.507548 0.512457 0.517364
```

**Screenshot:**

<img width="938" height="742" alt="Screenshot 2026-11-25 201045" src="https://github.com/user-attachments/assets/9b571c12-2de4-4363-899f-51cf9e2720cf" />

### Run 2

```
Average CPU execution time: 0.728520 ms
Average GPU kernel-only time: 0.031587 ms
Average GPU end-to-end time: 0.077320 ms
Kernel-only speedup: 23.063771x
End-to-end speedup: 9.422142x

Maximum absolute output difference: 0.000000
CPU and GPU outputs: MATCH (within tolerance)

First 10 GPU output values:
0.498196 0.503101 0.508006 0.512911 0.517814 0.497729 0.502638 0.507548 0.512457 0.517364
```

**Screenshot:**

<img width="810" height="687" alt="Screenshot 2026-11-25 201106" src="https://github.com/user-attachments/assets/57e0afb4-1909-4450-a932-c9bb85b286cb" />

### Summary table

| Metric | Run 1 | Run 2 |
|---|---|---|
| Average CPU time | 0.717745 ms | 0.728520 ms |
| Average GPU kernel-only time | 0.021043 ms | 0.031587 ms |
| Average GPU end-to-end time | 0.091440 ms | 0.077320 ms |
| Kernel-only speedup | 34.11x | 23.06x |
| End-to-end speedup | 7.85x | 9.42x |
| Max absolute difference (CPU vs GPU) | 0.000000 | 0.000000 |
| Output match | Yes | Yes |

---

## Analysis

- **Correctness:** The GPU output matches the CPU reference (maximum absolute difference 0.000000 at the printed precision), and the first 10 output values are identical across both runs. All three kernel tests passed.
- **Kernel-only speedup (23x to 34x):** This measures pure computation on the GPU. The massively parallel hardware is well suited to the matrix multiplication and element-wise activations in a neural network layer.
- **End-to-end speedup (about 8x to 9x):** This includes host-to-device and device-to-host memory transfers and launch overhead. The transfers dominate for a network this small, which is why the end-to-end gain is much lower than the kernel-only gain.
- **Run-to-run variation:** Kernel times in the sub-0.05 ms range are sensitive to clock states, driver scheduling and background desktop activity, so small differences between runs are expected. The GPU was in a low-power P5 state at idle, and it can take a moment to ramp up.
- **Scaling:** With larger batch sizes and layer widths, the compute portion grows faster than the transfer cost, so the end-to-end speedup would move closer to the kernel-only speedup.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `nvcc` not found | Confirm the toolkit is installed and `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v13.4\bin` is on `PATH`. Check with `where.exe nvcc`. |
| `Cannot find compiler 'cl.exe' in PATH` | Run `nvcc` from the Visual Studio Developer Command Prompt (`VsDevCmd.bat -arch=amd64 -host_arch=amd64`). |
| Kernel fails to launch or wrong results | Compile with the correct architecture flag. For this GPU use `-arch=sm_89`. |
| Driver CUDA version (13.3) lower than toolkit (13.4) | It worked in this experiment, but if you hit runtime errors, update the NVIDIA driver to the latest version. |
| Inconsistent timings | Run the benchmark several times, close GPU-heavy applications, and average the results. |

---

## Conclusion

The CUDA toolchain was set up and verified on an NVIDIA RTX 4500 Ada Generation GPU with CUDA Toolkit 13.4 and Visual Studio 2026 Build Tools. Neural network layers (matrix multiplication with bias, ReLU, and Sigmoid) were implemented as CUDA kernels, validated against a CPU implementation, and benchmarked. The GPU produced identical results and delivered roughly **23x to 34x** speedup for kernel execution and **8x to 9x** speedup end-to-end for the tested network size.

