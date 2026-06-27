

# Part 4: CUDA and Industry Terminology (Optional read only section- No assignment)

## 4.1 What CUDA Is, and Why It Exists

**CUDA (Compute Unified Device Architecture)** is NVIDIA's parallel computing platform and programming model, released in 2007. It extends C/C++ with a small set of new keywords and constructs that let a programmer describe massively parallel computation in a way that maps directly onto the SM / warp / thread hardware structure covered in Part 2.

CUDA's core contribution was making GPU computing accessible to ordinary programmers: instead of disguising your computation as a graphics operation (the pre-2007 GPGPU approach), you write code that looks recognizably like C, annotate which parts should run on the GPU, and let the CUDA compiler and runtime handle mapping that code onto thousands of hardware threads.

> **A note on the ecosystem:** CUDA is NVIDIA-specific. The vendor-neutral equivalent is **OpenCL**, and AMD's GPUs are programmed through **ROCm/HIP** (which is deliberately very similar to CUDA's syntax and concepts to ease porting). Higher-level deep learning frameworks like PyTorch and TensorFlow typically generate CUDA (or ROCm) calls automatically, so most AI practitioners today rarely write raw CUDA — but understanding it is what lets you read performance discussions, profiler output, and low-level documentation with real comprehension.

## 4.2 Host and Device: Two Separate Worlds

Every CUDA program is fundamentally a collaboration between two distinct processors with two distinct memory spaces:

- **Host:** The CPU and its system RAM. Runs the main program, handles I/O, and orchestrates the overall flow.
- **Device:** The GPU and its global memory. Executes the parallel, computationally heavy portions ("kernels").

```
┌───────────────────────┐         ┌───────────────────────┐
│        HOST (CPU)      │         │       DEVICE (GPU)     │
│                        │         │                        │
│   Host code runs here   │         │   Kernel code runs here │
│   System RAM            │  PCIe   │   GPU Global Memory     │
│                        │ ◄─────► │                        │
└───────────────────────┘  bus    └───────────────────────┘
```

A typical CUDA program follows this sequence:

1. **Allocate** memory on both host and device.
2. **Copy** input data from host memory to device memory (`cudaMemcpy`).
3. **Launch** a kernel — the GPU function — specifying how many threads to run.
4. The GPU executes the kernel across thousands of threads in parallel.
5. **Copy** the results back from device memory to host memory.
6. **Free** allocated memory.

## 4.3 Kernels and Launch Configuration

A **kernel** is a function written to be executed by many GPU threads simultaneously, each operating on its own piece of data. In CUDA, you mark a kernel with the `__global__` qualifier.

```c
// A CUDA kernel for element-wise vector addition: C[i] = A[i] + B[i]
__global__ void vectorAdd(const float *A, const float *B, float *C, int N) {
    // Each thread computes its own unique global index
    int i = blockIdx.x * blockDim.x + threadIdx.x;

    // Guard against threads beyond the array bounds
    if (i < N) {
        C[i] = A[i] + B[i];
    }
}
```

Notice something important, consistent with the SIMT model from Part 2: **you write the logic for a single thread**. There is no explicit loop over the array — the "loop" is implicit, expressed by launching one thread per array element. Every one of the thousands of threads executes this exact same function body, differing only in the value of `i`.

### 4.3.1 Launch Configuration: Grid and Block Dimensions

To run this kernel, the host code specifies the **launch configuration** — how many blocks, and how many threads per block:

```c
int N = 1 << 20;              // ~1 million elements
int threadsPerBlock = 256;
int blocksPerGrid = (N + threadsPerBlock - 1) / threadsPerBlock; // ceiling division

// Triple angle-bracket syntax launches the kernel on the GPU
vectorAdd<<<blocksPerGrid, threadsPerBlock>>>(d_A, d_B, d_C, N);
```

This launches `blocksPerGrid × threadsPerBlock` total threads — roughly one per array element, organized into blocks of 256 threads each. The hardware then schedules these blocks onto available SMs, and within each block, groups of 32 threads execute together as warps (Part 2.3.2).

```
N = 1,048,576 elements, threadsPerBlock = 256
            │
            ▼
blocksPerGrid = ceil(1,048,576 / 256) = 4,096 blocks

         GRID (4,096 blocks)
   ┌────────────────────────────┐
   │ Block 0  Block 1  ... Block 4095 │
   └────────────────────────────┘
         each Block = 256 threads
         = 8 warps of 32 threads each
```

### 4.3.2 A Slightly Richer Example: Using Shared Memory

To connect back to Part 3, here is a simplified pattern showing why shared memory matters — a 1D "tile and reuse" sketch, conceptually similar to the core idea behind matrix multiplication kernels:

```c
__global__ void sumWithSharedMemory(const float *input, float *output, int N) {
    __shared__ float tile[256];   // Fast, block-local scratchpad

    int tid = threadIdx.x;
    int i = blockIdx.x * blockDim.x + tid;

    // Each thread loads ONE element from slow global memory into fast shared memory
    tile[tid] = (i < N) ? input[i] : 0.0f;
    __syncthreads();   // Barrier: wait until ALL threads in the block have loaded

    // Now threads can repeatedly reuse 'tile' from fast shared memory
    // instead of going back to slow global memory multiple times
    // ... (e.g., a parallel reduction would happen here) ...
}
```

`__syncthreads()` is a **block-level barrier**: every thread in the block must reach this point before any thread is allowed to proceed past it. This is the basic synchronization primitive that makes shared-memory cooperation between threads in the same block safe.

## 4.4 The Development Flow

At a high level, the practical workflow for CUDA development is:

1. **Write** host (CPU) code and device (GPU kernel) code, typically in the same `.cu` file.
2. **Compile** with NVIDIA's `nvcc` compiler, which separates host and device code, compiling each appropriately.
3. **Run** the resulting executable on a system with a CUDA-capable GPU.
4. **Profile** using tools like NVIDIA Nsight Systems / Nsight Compute, which report exactly the metrics discussed in Part 3 — occupancy, memory throughput, coalescing efficiency, warp divergence — making the abstract concepts from Part 3 directly visible and measurable.
5. **Debug and iterate**, often guided directly by profiler output rather than guesswork.

---

## 4.5 Industry Terminology Reference

This section contains some key terminologies you might want to know!

| Term | Plain-Language Definition | Where You'll See It |
|---|---|---|
| **Kernel** | A function written to run on the GPU across many parallel threads | "We optimized the convolution kernel for better throughput." |
| **Thread** | The smallest unit of GPU execution; processes one data element | "Each thread handles one pixel." |
| **Block (Thread Block)** | A group of threads scheduled together on one SM, can share fast memory | "We tuned the block size to 256 for better occupancy." |
| **Grid** | The full set of blocks covering an entire kernel launch | "The grid covers the whole image." |
| **Warp** | A hardware-scheduled group of 32 threads executing in lockstep (NVIDIA term; AMD: wavefront, 64 threads) | "Warp divergence hurt our performance by 3×." |
| **SM (Streaming Multiprocessor)** | A GPU's core processing unit, containing many simple cores plus shared memory and schedulers (AMD: Compute Unit) | "This GPU has 132 SMs." |
| **SIMT** | Single Instruction, Multiple Threads — the GPU's execution model where many threads run identical code in lockstep | "GPUs use SIMT, not SIMD, as their execution model." |
| **Occupancy** | The fraction of a GPU's maximum thread/warp capacity actively in use | "Low occupancy was limiting our latency hiding." |
| **Throughput** | The total amount of work completed per unit time (vs. latency, the time for one task) | "GPUs are optimized for throughput, not latency." |
| **Latency Hiding** | Switching to other ready warps while one warp waits on slow memory, to keep cores busy | "The scheduler hides memory latency by switching warps." |
| **Memory Bandwidth** | The rate at which data can move between memory and compute (GB/s or TB/s) | "This GPU has 3 TB/s of memory bandwidth." |
| **Coalescing** | Combining multiple threads' memory accesses into fewer, wider transactions | "Uncoalesced access tanked our bandwidth utilization." |
| **Shared Memory** | Fast, programmer-managed memory local to a thread block | "We tiled the matrix multiply using shared memory." |
| **Global Memory** | The GPU's large, chip-wide DRAM (GDDR6X/HBM), visible to all threads | "Input data is copied into global memory first." |
| **Divergence (Warp Divergence)** | Performance loss when threads in the same warp take different branch paths and must serialize | "Branch-heavy code causes warp divergence." |
| **Synchronization (`__syncthreads()`)** | A barrier ensuring all threads in a block reach a point before any proceed | "We added a sync point before the reduction step." |
| **Tensor Core** | A specialized hardware unit for fast mixed-precision matrix multiply-accumulate, used heavily in AI workloads | "FP16 training is accelerated by Tensor Cores." |
| **Compute Capability** | NVIDIA's versioning system describing which hardware features a given GPU generation supports | "This library requires compute capability 7.0 or higher." |
| **Host / Device** | Host = CPU and its memory; Device = GPU and its memory | "Copy the result from device to host." |
| **CUDA Core** | An individual simple ALU within an SM (not the same as a CPU core — far simpler, only useful in large numbers) | "This SM contains 128 CUDA cores." |

