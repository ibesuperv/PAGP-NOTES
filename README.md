# Parallel Architecture and GPU Programming (PAGP) — Notes & Exam Solutions

Comprehensive lecture notes and exam question bank solutions for **Parallel Architecture and GPU Programming (PAGP)** (7th Semester B.Tech Computer Science and Engineering), structured according to *Computer Architecture: A Quantitative Approach* (Hennessy & Patterson, 6th Edition) and the NVIDIA CUDA C Programming Guide.

---

## Repository Structure

### 📁 [UNIT-1: Instruction-Level Parallelism & Processor Architecture](./UNIT-1)
1. **[01. Historical Trends and the Multicore Shift](./UNIT-1/01_historical_trends_and_the_multicore_shift.md)**
2. **[02. Classes of Parallelism and Flynn's Taxonomy](./UNIT-1/02_classes_of_parallelism_and_flynns_taxonomy.md)**
3. **[03. Defining Computer Architecture and ISA](./UNIT-1/03_defining_computer_architecture_and_isa.md)**
4. **[04. Dependability and Performance Metrics](./UNIT-1/04_dependability_and_performance_metrics.md)**
5. **[05. Quantitative Principles and the CPU Performance Equation](./UNIT-1/05_quantitative_principles_and_cpu_performance_equation.md)**
6. **[06. Fundamentals of Pipelining and the Classical RISC Pipeline](./UNIT-1/06_fundamentals_of_pipelining_and_risc_pipeline.md)**
7. **[07. Pipeline Hazards and Solutions](./UNIT-1/07_pipeline_hazards_and_solutions.md)**
8. **[08. Instruction-Level Parallelism (ILP) and Dependences](./UNIT-1/08_instruction_level_parallelism_and_dependences.md)**
9. **[09. Compiler Techniques for ILP and Loop Unrolling](./UNIT-1/09_compiler_techniques_for_ilp_and_loop_unrolling.md)**
10. **[10. Advanced Dynamic Branch Prediction and Intel Core i7](./UNIT-1/10_advanced_dynamic_branch_prediction_and_intel_core_i7.md)**
11. **[11. Dynamic Scheduling and Tomasulo's Algorithm](./UNIT-1/11_dynamic_scheduling_and_tomasulo_algorithm.md)**

---

### 📁 [UNIT-2: Multiprocessors, Cache Coherence & Consistency](./UNIT-2)
1. **[01. Performance of Symmetric Shared-Memory Multiprocessors](./UNIT-2/01_performance_of_symmetric_shared_memory_multiprocessors.md)** (Section 5.3)
2. **[02. Distributed Shared-Memory and Directory-Based Coherence](./UNIT-2/02_distributed_shared_memory_and_directory_based_coherence.md)** (Section 5.4, with FSM state diagrams)
3. **[03. Multiprocessor Synchronization: The Basics](./UNIT-2/03_multiprocessor_synchronization_the_basics.md)** (Section 5.5: Atomic primitives, `lr`/`sc`, TTAS spinlocks, 8-step bus trace)
4. **[04. Models of Memory Consistency](./UNIT-2/04_models_of_memory_consistency.md)** (Section 5.6: Sequential Consistency, DRF, TSO, PSO, WO, Release Consistency, fences)
5. **[05. Unit 2 Master Exam Question Bank & Model Solutions](./UNIT-2/05_unit2_exam_question_bank_solutions.md)** (29 fully-solved university exam questions)

---

### 📁 [UNIT-3: Data-Level Parallelism, Vector Architectures & GPU Programming](./UNIT-3)
1. **[01. Unit 3 Exam Question Bank Solutions](./UNIT-3/01_unit3_exam_question_bank_solutions.md)** (Gather/Scatter, Vector Chaining, Loop-carried dependence, Warps, CUDA Vector Add)
2. **[02. Vector Architecture & Loop Parallelism Solutions](./UNIT-3/02_unit3_vector_architecture_and_loop_parallelism_solutions.md)** (Vector lanes, GCD test, Convoys & chimes, VMIPS architecture)
3. **[03. Vector Memory & GPU Branching Solutions](./UNIT-3/03_unit3_vector_memory_and_gpu_branching_solutions.md)** (Stride-1 vs. Stride-32 bank latencies, VLR/VMR, Fermi innovations)
4. **[04. Vector, GPU & Thread Scheduling Solutions](./UNIT-3/04_unit3_vector_gpu_and_thread_scheduling_solutions.md)** (VMIPS ISA, DAXPY MIPS vs. VMIPS, SIMD thread scheduling)
5. **[05. SIMD Thread Scheduling & VMIPS Innovations](./UNIT-3/05_unit3_simd_thread_scheduling_and_vmips_innovations.md)** (OpenCL NDRange, Scalar register justification, Vectorization)
6. **[06. CUDA Processing Flow, Matrix Multiplication & DAXPY](./UNIT-3/06_unit3_cuda_processing_flow_matrix_mul_and_daxpy_solutions.md)** (CUDA Host-to-Device flow, full CUDA C Matrix Multiplication program)
7. **[07. Gather-Scatter, CUDA DAXPY & VMIPS Solutions](./UNIT-3/07_unit3_gather_scatter_cuda_daxpy_and_vmips_solutions.md)** (Sparse matrices, 256 threads/block CUDA kernel)
8. **[08. Matrix Multiplication CPU to GPU & DAXPY Solutions](./UNIT-3/08_unit3_matrix_multiplication_cpu_to_gpu_and_daxpy_solutions.md)** (Shared memory scope, CPU host vs. GPU device matrix multiplication)
9. **[09. SIMD Multimedia, CPU vs. GPU & Matrix Solutions](./UNIT-3/09_unit3_simd_multimedia_cpu_vs_gpu_and_matrix_solutions.md)** (CPU vs. GPU architectural differences, SIMD extensions MMX/SSE/AVX)
10. **[10. GPU Memory, Warp Divergence & Architectural Trade-offs](./UNIT-3/10_unit3_gpu_memory_warp_divergence_and_architectural_tradeoffs.md)** (Shared, constant, texture memory, warp divergence mitigation, Vector vs. SIMD vs. GPU trade-offs)
