# From CPUs to GPUs: Parallelism, Tensors, CUDA, PyTorch & Big Data Analogy

## Introduction

During my bachelor’s degree, I studied the 8086 microprocessor—its architecture, registers, ALU, memory interactions, and assembly instructions. That experience grounded me in how computer operations translate into processor behavior at the electronic circuit level.

After entering the software industry, my work shifted into the data engineering and database domain—designing, querying, and optimizing systems that used **RDBMS** and later **Big Data ecosystems** like Hadoop and Spark. That transition taught me how scaling data and computation fundamentally changes system architecture and execution models.

As the AI and ML revolution accelerated, especially with deep learning, I wanted to understand what makes **GPUs**, **CUDA**, **tensors**, and **PyTorch** so central. To make sense of it, I naturally connected back to what I already knew: CPU architecture, instruction execution, and parallel data processing.

In that journey, I realized something interesting—there is a strong analogy between:

* the transition from **CPUs → GPUs**, and
* the shift from **RDBMS → Big Data processing**

Both transitions were driven by the need for scale—more data, more computation, and more parallelism. This blog is an attempt to connect those worlds and explain GPU computing in a way that feels familiar to anyone who has worked with traditional CPUs or large-scale data sys

In this blog, tried to explain at high level CPU vs GPU architectures, how parallelism works, and how it connects to high-level frameworks like PyTorch and Big Data processing.

## 1. CPU vs GPU: Key Differences

### CPU Architecture

A CPU is designed for general-purpose computing with a focus on low-latency execution and complex instruction handling. Key components include:

* **ALU (Arithmetic Logic Unit):** Performs arithmetic and logic operations.
* **Control Unit:** Fetches, decodes, and executes instructions sequentially.
* **Registers:** Small, ultra-fast storage for operands and instructions (e.g., AX, BX in 8086).
* **Cache Memory:** High-speed memory close to the core to reduce access latency.
* **RAM Access:** Reads/writes data from main memory when necessary.

**Characteristics:**

* Few powerful cores (1–16 in modern CPUs).
* Optimized for sequential execution and complex branching.
* Excellent for logic-heavy tasks and low-latency operations.

Further reading: [CPU Architecture Basics](https://en.wikipedia.org/wiki/Central_processing_unit), [8086 Microprocessor](https://en.wikipedia.org/wiki/Intel_8086)

### GPU Architecture

A **GPU (Graphics Processing Unit)** was originally designed to accelerate rendering of graphics on screens, but modern GPUs are highly effective for **parallel numeric computations** used in scientific computing and deep learning.

* **GPU Cores (called CUDA cores on NVIDIA GPUs) / Stream Processors:** Thousands of simple arithmetic units that execute math operations in parallel. These are hardware components—different from CUDA the software platform. Thousands of simple cores that execute arithmetic operations in parallel.
* **Streaming Multiprocessors (SMs):** Groups of CUDA cores sharing control logic and memory resources.
* **Registers per Thread:** Small memory space for each thread.
* **Shared Memory per SM:** Fast memory shared among threads within a block.
* **Global Memory:** Large, slower memory accessible to all threads across SMs.
* **Warp Scheduler:** Hardware unit that schedules 32 threads (a warp) to execute instructions simultaneously.

**Characteristics:**

* Thousands of lightweight cores.
* Optimized for SIMD/SIMT execution (same instruction, multiple data elements).
* Excellent for tasks like matrix operations, image processing, and neural network computations.

Further reading: [GPU Architecture](https://www.vmware.com/docs/exploring-the-gpu-architecture), [CUDA Programming](https://developer.nvidia.com/cuda-zone)

### CUDA Explained

**CUDA (Compute Unified Device Architecture)** is NVIDIA's **parallel computing software platform and programming model**—not hardware. It allows developers to write programs that run on GPU hardware cores.

CUDA provides:

* APIs and libraries for GPU programming
* Memory management between CPU and GPU
* Ability to write **kernels**, functions executed in parallel on GPU threads

So:

* **CUDA cores = hardware units inside the GPU**
* **CUDA = software platform used to program those cores**

Further reading: [What is CUDA?](https://developer.nvidia.com/cuda-zone)

[CUDA Course](https://github.com/Infatoshi/cuda-course)


## 2. From 8086 to Tensors, CUDA Kernels and PyTorch

### 8086 Example

```asm
MOV AX, 5
MOV BX, 7
ADD AX, BX
```

* Executes **one instruction at a time**

### Adding Arrays Sequentially (CPU Style)

```c
for (int i=0; i<N; i++) {
    C[i] = A[i] + B[i];
}
```

* CPU executes **one addition at a time**
* 
### Introducing Tensors — The Foundation of Modern GPU Computing

Before understanding CUDA, GPUs, and deep learning frameworks, we must first understand tensors.

### What Is a Tensor?

A **tensor** is a generalized multi-dimensional numerical data structure.

### Why Do We Need Tensors When Matrices Already Exist?

1. **Real AI/ML Data Is Multi-Dimensional**
   Matrices are only 2D—most real workloads are not.
   Example: a video → 5D tensor (batch × frames × height × width × channels)

2. **Device Awareness (CPU or GPU)**
   Tensors internally track whether they live in CPU memory or GPU memory—matrices don’t.

3. **Automatic Differentiation Required for Training**
   Tensors carry gradient history, enabling backpropagation.
   A normal matrix cannot store computation graphs.

4. **Unified Numeric Representation**
   Tensors contain metadata: shape, dtype, stride, layout—allowing optimized computation.

5. **Built for Parallelism**
   Tensor libraries efficiently map operations to vectorized CPU instructions or thousands of GPU threads.

**Summary:**
Matrices are great—but tensors scale numerical computing for machine learning, GPUs, and high-dimensional data.

---

## CUDA, CUDA Kernels & PyTorch — How They Connect

Now that tensors represent our data, how do we compute with them efficiently?

### What Is CUDA?

**CUDA (Compute Unified Device Architecture)** is NVIDIA’s **software platform and programming model** that allows general-purpose computation on GPUs.

* Provides APIs, compilers, libraries
* Manages GPU memory and execution
* Enables launching parallel tasks

CUDA does **not** mean “GPU core”—it is the **software ecosystem** used to program GPUs.

### What Are CUDA Cores?

Inside an NVIDIA GPU are thousands of lightweight arithmetic units called **CUDA cores**—the hardware that performs computation.

* CPU: few, powerful cores
* GPU: many simple cores working together

Tensors stored on a GPU are computed using these cores.

### What Is a CUDA Kernel?

A **kernel** is a function written to run on the GPU.

* Defines the computation each GPU thread performs
* Executes in parallel across thousands of threads

Example — Add two arrays:

```cpp
__global__ void add(int *A, int *B, int *C) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    C[i] = A[i] + B[i];
}
```

A single kernel launch can compute millions of additions at once.

### Where Does PyTorch Fit In?

(PyTorch)[https://pytorch.org/] is a **high-level machine learning framework** that:

* Creates and stores **tensors**
* Moves tensors to CPU or GPU: `tensor.to('cuda')`
* Automatically launches optimized CUDA kernels internally

Example:

```python
import torch
A = torch.randn(1000, 1000, device='cuda')
B = torch.randn(1000, 1000, device='cuda')
C = A + B  # PyTorch triggers a CUDA kernel
```

You write simple math—PyTorch selects, schedules, and executes kernels on the GPU.

---

### The Relationship in One Sentence:

**Tensors store data → CUDA kernels define operations → CUDA runs kernels on GPU cores → PyTorch automates the entire process.**


---

## 3. Matrix Multiplication Example

### Sequential CPU Implementation

```python
for i in range(N):
    for j in range(N):
        C[i][j] = 0
        for k in range(N):
            C[i][j] += A[i][k] * B[k][j]
```

### GPU Implementation (CUDA Kernel)

```cpp
__global__ void matMul(float *A, float *B, float *C, int N) {
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;

    if(row < N && col < N){
        float sum = 0;
        for(int k=0; k<N; k++){
            sum += A[row*N + k] * B[k*N + col];
        }
        C[row*N + col] = sum;
    }
}
```

* Each GPU thread computes **one element of the result matrix**
* Kernels map threads to tensor elements efficiently

### Rough Timing Estimates

* CPU sequential: ~1 billion operations → ~1 second (simplified)
* GPU with 1024 cores: ~1 ms – 10 ms

---

## 4. How GPU Achieves Parallel Computing

### Hardware Level

* **Streaming Multiprocessors (SMs)**: Contain 64–128 CUDA cores each
* **Registers per thread** and **shared memory per SM**
* Threads executed in **warps (32 threads)**
* Hardware handles **scheduling and latency hiding**

### Software Level

* CUDA kernel = software interface
* Each thread is automatically mapped to a CUDA core
* Programmer defines operations; hardware executes in parallel

Further reading: [CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html)

---

## 5. Why a Collection of CPUs Can't Replace a GPU

1. **Core count limitation:** CPUs are few; GPUs have thousands of cores.
2. **Memory architecture:** GPUs are designed for high-throughput parallel access.
3. **Instruction simplicity:** GPU cores are simpler and more efficient for repeated operations.
4. **Scheduling & latency hiding:** GPUs handle thousands of threads efficiently.
5. **Energy efficiency:** Scaling CPUs to match GPU cores is impractical.

---

## 6. Analogy to Big Data Processing

| Concept                    | CPU / RDBMS                         | GPU / Big Data Processing                            |
| -------------------------- | ----------------------------------- | ---------------------------------------------------- |
| Cores / Workers            | Few, powerful                       | Many, simple                                         |
| Execution                  | Sequential / limited parallelism    | Massively parallel (SIMD / distributed)              |
| Best for                   | Complex logic, small-to-medium data | Repeated operations on large datasets                |
| Hardware/software relation | CPU executes instructions directly  | GPU cores / Hadoop workers execute tasks in parallel |
| Memory                     | Large cache / RAM                   | Shared memory / distributed storage                  |
| Programming                | SQL / imperative programming        | MapReduce / Spark / CUDA kernel                      |

**Example:** Summing a large dataset

* RDBMS / CPU: sequential sum → slow
* GPU / Big Data: parallel sum → fast

