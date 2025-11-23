# From CPUs to GPUs/TPUs: Parallelism, Tensors, CUDA, PyTorch/TensforFlow

## Introduction

Two LinkedIn posts recently piqued my curiosity—one about **NVIDIA’s CUDA** and another about **Google owning the full AI stack**.

Before diving into the details, here’s a little bit of my context: during my bachelor’s, I studied the **8086 microprocessor**—its architecture, registers, ALU, memory interactions, and assembly instructions. That gave me a solid grounding in how instructions translate into processor behavior at the circuit level. Later, my career shifted to **data engineering and databases**, working with RDBMS and eventually **Big Data frameworks** like Hadoop and Spark. This taught me how scaling data and computation fundamentally changes system architecture and execution models.

This blog is my attempt to gather basic detals on **CPU vs GPU/TPU architectures** and explain how they relate to high-level frameworks like **PyTorch and TensorFlow**. Also wanted to share an interesting analogy i observed while gathering details about **CPU vs GPU/TPU architectures**. 

* The transition from **CPUs → GPUs** parallels
* The shift from **RDBMS → Big Data processing**
Both transitions are driven by **scale**—more data, more computation, and more parallelism.

## 1. CPU vs GPU: Key Differences

### CPU Architecture

A CPU is designed for general-purpose computing with a focus on low-latency execution and complex instruction handling. Key components include:

* **ALU (Arithmetic Logic Unit):** Performs arithmetic and logic operations.
* **Control Unit:** Fetches, decodes, and executes instructions sequentially.
* **Registers:** Small, ultra-fast storage for operands and instructions (e.g., AX, BX in 8086).
* **Cache Memory:** High-speed memory close to the core to reduce access latency.
* **RAM Access:** Reads/writes data from main memory when necessary.

Further reading: [CPU Architecture Basics](https://en.wikipedia.org/wiki/Central_processing_unit), [8086 Microprocessor](https://en.wikipedia.org/wiki/Intel_8086)

### GPU Architecture

A **GPU (Graphics Processing Unit)** was originally designed to accelerate rendering of graphics on screens, but modern GPUs are highly effective for **parallel numeric computations** used in scientific computing and deep learning.

* **GPU Cores (called CUDA cores on NVIDIA GPUs) / Stream Processors:** Thousands of simple arithmetic units that execute math operations in parallel. These are hardware components—different from CUDA the software platform. Thousands of simple cores that execute arithmetic operations in parallel.
* **Streaming Multiprocessors (SMs):** Groups of CUDA cores sharing control logic and memory resources.
* **Registers per Thread:** Small memory space for each thread.
* **Shared Memory per SM:** Fast memory shared among threads within a block.
* **Global Memory:** Large, slower memory accessible to all threads across SMs.
* **Warp Scheduler:** Hardware unit that schedules 32 threads (a warp) to execute instructions simultaneously.


Further reading: [CUDA Programming](https://developer.nvidia.com/cuda-zone) ,[GPU Architecture](https://www.vmware.com/docs/exploring-the-gpu-architecture), 

### TPU Architecture

A **TPU (Tensor Processing Unit)** is Google’s custom chip designed specifically for **high-throughput tensor/matrix computations** common in deep learning, rather than general-purpose computation.

* **Matrix Multiply Unit (MXU) / Systolic Array:** Large arrays of multiply-accumulate units that perform massive matrix multiplications in parallel. The heart of the TPU.
* **Vector Processing Unit (VPU):** Handles element-wise operations and vector math outside of the MXU.
* **High-Bandwidth Memory (HBM):** On-chip memory for storing weights, activations, and intermediate results; much faster than off-chip DRAM.
* **Scalar Unit / CPU-like cores:** Manage control flow, orchestrate data movement, and run non-matrix operations.
* **Infeed / Outfeed Queues:** Hardware pipelines that stream data into and out of the TPU efficiently.
* **Interconnect:** For multi-TPU setups (pods), a high-speed mesh network connects TPU chips for distributed computation.

Further reading: [TPU Architecture Overview](https://cloud.google.com/tpu/docs/system-architecture), [Inside Google’s TPU](https://cloud.google.com/blog/products/ai-machine-learning/under-the-hood-of-googles-tensor-processing-units-tpus)

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
[Nvidia Developer : Accelerating Applications with Parallel Algorithms](https://youtu.be/Sdjn9FOkhnA?si=k1I5wKSLeQGBw8Kk)


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

### Introducing Tensors — The Foundation of Modern GPU Computing

Before understanding CUDA, GPUs, and deep learning frameworks, we must first understand tensors.

### What Is a Tensor?

A **tensor** is a generalized multi-dimensional numerical data structure.

### Why Do We Need Tensors When Matrices Already Exist?

**Device Awareness (CPU or GPU)**
   Tensors internally track whether they live in CPU memory or GPU memory—matrices don’t.

**Automatic Differentiation Required for Training**
   Tensors carry gradient history, enabling backpropagation.
   A normal matrix cannot store computation graphs.

**Unified Numeric Representation**
   Tensors contain metadata: shape, dtype, stride, layout—allowing optimized computation.

**Built for Parallelism**
   Tensor libraries efficiently map operations to vectorized CPU instructions or thousands of GPU threads.


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

## TensorFlow, XLA & TPUs — How They Connect


### What Is a TPU?

A [TPU (Tensor Processing Unit)](https://cloud.google.com/tpu) is Google’s custom chip built for deep-learning workloads.

* Specialized matrix-multiply hardware
* High throughput, low power
* Available through Google Cloud

Designed specifically for neural networks—not general computing.Architecture: [https://cloud.google.com/tpu/docs/system-architecture](https://cloud.google.com/tpu/docs/system-architecture)

### What Is XLA?

[XLA (Accelerated Linear Algebra)](https://www.tensorflow.org/xla) is a compiler that optimizes tensor computations before execution.

* Fuses operations for faster execution
* Generates device-specific machine code
* Targets CPUs, GPUs, and TPUs

Think of XLA as TensorFlow’s optimization and translation layer.Architecture: [https://www.tensorflow.org/xla/architecture](https://www.tensorflow.org/xla/architecture)

### What Is TensorFlow?

[TensorFlow](https://www.tensorflow.org/) is Google’s open-source ML framework for building and training neural networks.

* Creates tensors and operations
* Runs on CPU, GPU, or TPU
* Provides high-level APIs like Keras

TensorFlow is the interface—not the hardware.Guide: [https://www.tensorflow.org/guide](https://www.tensorflow.org/guide)

### How They Work Together

1. TensorFlow builds the computation graph
2. XLA compiles and optimizes it
3. The TPU executes the compiled program

You write model code—TensorFlow + XLA handle hardware execution.TPU usage guide: [https://www.tensorflow.org/guide/tpu](https://www.tensorflow.org/guide/tpu)



### One-Sentence Relationship

**TensorFlow defines the model → XLA compiles it → TPUs run it efficiently.**

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

### TPU Implementation

```
import tensorflow as tf
import numpy as np

# Initialize TPU
resolver = tf.distribute.cluster_resolver.TPUClusterResolver()
tf.tpu.experimental.initialize_tpu_system(resolver)
strategy = tf.distribute.TPUStrategy()

# Random matrices
N = 1024
A = tf.constant(np.random.randn(N, N), dtype=tf.float32)
B = tf.constant(np.random.randn(N, N), dtype=tf.float32)

with strategy.scope():
    @tf.function  # Compiles for TPU via XLA
    def matmul_tpu(A, B):
        return tf.matmul(A, B)

C = matmul_tpu(A, B)
print(C)
```
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

* the transition from **CPUs → GPUs**, and
* the shift from **RDBMS → Big Data processing**

Both transitions were driven by the need for scale—more data, more computation, and more parallelism. 

| Concept                    | CPU / RDBMS                         | GPU / Big Data Processing                            |
| -------------------------- | ----------------------------------- | ---------------------------------------------------- |
| Cores / Workers            | Few, powerful                       | Many, simple                                         |
| Execution                  | Sequential / limited parallelism    | Massively parallel (SIMD / distributed)              |
| Best for                   | small-to-medium data | large datasets                |
| Memory                     | Large cache / RAM                   | Shared memory / distributed storage                  |



