# Introduction

High-performance scientific computing relies heavily on the ability of software to efficiently utilise modern CPU architectures. As processor generations evolve-introducing wider vector units, expanded core counts, heterogeneous memory hierarchies, and new instruction sets-the performance of large scientific applications is increasingly determined by compiler quality and the effectiveness of architecture-specific optimisations. Benchmarking compilers across multiple CPU families is therefore essential for understanding how scientific workloads behave on modern HPC systems and for guiding users toward best practices.

The Modules for Experiments in Stellar Astrophysics (MESA) code represents a demanding and widely used benchmark for such studies. Written primarily in Fortran, MESA performs computationally intensive stellar evolution simulations with substantial floating-point workloads and OpenMP-based node-level parallelism. By combining intensive mathematical computations with high-performance data handling, it serves as an ideal benchmark for evaluating compilers on real-world scientific workloads.

Stars can exist and evolve in a number of distinctive states and MESA aims to be able to simulate as many of them as are feasible. During these different states, the simulations will vary in a number of computationally significant ways, including the appropriate resolution in time and space and how the simulation is expected to scale with increasing numbers of CPU cores. We experimented using a number of different integration tests that represent different kinds of stars. Though this particular selection is unlikely to have exercised the full range of MESA's capabilities, the cases were selected to include some of the most computationally intensive.

This work investigates the performance of GCC (gfortran) and the Intel Fortran Compiler (ifx) across several CPU architectures available on the University of Birmingham's BlueBEAR HPC cluster-Ice Lake, Sapphire Rapids, and Emerald Rapids-as well as on Lenovo's LENOX system with Granite Rapids processors. Because MESA does not provide Intel-specific Makefile options by default, a custom build configuration was developed to ensure that both compilers are tested with fair and equivalent optimisation strategies. Particular attention was paid to using native builds on each target CPU, ensuring each compiler has full access to platform-specific features such as AVX‑512 variants and tuning heuristics.

The goal of this study is to present a transparent, reproducible, and architecture‑aware comparison of compiler performance for MESA across a diverse range of current Intel microarchitectures. The results aim to guide HPC users, system administrators, and researchers in choosing optimal compiler configurations for scientific workflows on current x86-based platforms.

# Methodology

## Benchmark Platforms

Performance tests were conducted on four Intel CPU architectures:

- **Ice Lake** (BlueBEAR): dual-socket nodes with 2 × 36-core Intel Xeon Platinum 8360Y CPUs (72 cores total).
- **Sapphire Rapids** (BlueBEAR): dual-socket nodes with 2 × 56-core Intel Xeon Platinum 8480CL CPUs (112 cores total).
- **Emerald Rapids** (BlueBEAR): dual-socket nodes with 2 × 56-core Intel Xeon Platinum 8570 CPUs (112 cores total).
- **Granite Rapids** (LENOX): dual-socket nodes with 2 × 128-core Intel Xeon 6 6980P CPUs (256 cores total).

MESA is parallelised using shared-memory OpenMP only; no distributed-memory (MPI) parallelism is employed. All benchmarks were therefore run on a single compute node per job.

### Software Stack

On the BlueBEAR cluster, the following compiler modules were used:

- GNU Fortran: **gfortran 13.3.0**
- Intel Fortran (LLVM-based): **ifx 2024.2.0**

On the LENOX cluster, the corresponding modules were:

- GNU Fortran: **gfortran 14.2.0**
- Intel Fortran (LLVM-based): **ifx 2025.2.1**

These versions were selected as the most recent production compilers available via the respective module systems at the time of the study.

## MESA Configuration and Test Selection

### Build Configuration

All MESA builds used a common configuration intended to reflect a typical production setup while keeping the benchmark focused on the core stellar evolution kernels. The following options were used:

```
PROFILE=release
WITH_OPENMP=yes
WITH_ADIPLS=no
WITH_CRLIBM=no
WITH_GYRE=no
WITH_PGSTAR=no
```

MESA itself is written primarily in Fortran and uses OpenMP for intra-node parallelism. Because MESA does not include Intel-targeted build rules by default, a new Makefile configuration was created for the Intel ifx compiler. This ensured:

- correct handling of Intel Fortran syntax and modules,
- consistent linking behaviour across compilers,
- parity between GCC and Intel build options (including OpenMP and optimisation flags).

We are working with the MESA maintainers to make this configuration available upstream for potential inclusion in the MESA mainline.

### Overview of MESA Integration Tests

Longer MESA integration tests are designed so that the final, critical phase of the evolution can be run in isolation from the the rest of the test, which is regarded as only being necessary to prepare the input for the actual test. Whether a test runner includes the whole simulation or just the final critical part is controlled by the parameter `MESA_SKIP_OPTIONAL`. If this parameter is set (to any value), only the final part of the simulation is executed. To focus on the most computationally demanding phases and keep overall runtimes manageable, all benchmarks in this study were run with `MESA_SKIP_OPTIONAL` enabled, thereby executing only the critical final stage of each integration test.

To keep the runtime of the whole test suite reasonable, MESA's integration tests are generally configured at lower spatial and temporal resolutions than researchers would use for full scientific studies. However, the relative computational intensity between different phases and test cases is maintained, making them suitable for comparative performance analysis.

### Selected Benchmark Cases

We selected four MESA integration tests that collectively span a range of stellar types and evolutionary phases, with an emphasis on computationally challenging regimes:

1. `wd_stable_h_burn`
2. `20M_pre_ms_to_core_collapse`
3. `15M_dynamo`
4. `5M_cepheid_blue_loop`

Each of these tests corresponds to a physically distinct scenario and stresses MESA in different ways, as described below.

#### wd_stable_h_burn

White dwarfs are very dense stars that are the remnant cores of stars up to about eight times as massive as the Sun. They sometimes exist near enough to companion stars that some of the companion (which is still mostly hydrogen) steadily settles on the white dwarf's surface and it can begin to shine by fusing the hydrogen into helium. This test case approximates such a situation with parameters that are finely tuned to produce steady fusion of hydrogen into helium. The extreme density of the white dwarf and thinness of the hydrogen layer make this simulation computationally demanding.

#### 20M_pre_ms_to_core_collapse

This integration test simulates the life of a star 20 times as massive as the Sun from its birth to the moments before it explodes as a supernova (which MESA itself does not simulate). Pre-supernova evolution is generally one of the most computationally intensive phases, as it requires a high-resolution mesh with many chemical species under extreme physical conditions. For this benchmark we used only the final stage of this otherwise very long integration test, which follows the star from when silicon begins to fuse into even heavier elements (around 4 billion kelvin) until the central parts of the star have begun to collapse.

#### 15M_dynamo

MESA is able to approximately include changes in a star's structure and evolution caused by rotation. This test case serves to check that the additional computations required in this mode are functioning correctly by simulating a star 15 times as massive as the Sun from birth until about halfway through a late stage in which helium fuses into carbon and oxygen in the core.

#### 5M_cepheid_blue_loop

This integration test follows a star 5 times as massive as the Sun as it fuses both helium in the core and hydrogen in a thin shell around the helium core. The increasing power from the core interacts with the hydrogen shell and is known to require enough spatial resolution around the shell to correctly follow its behaviour. Our benchmarking is only based on the second part of the test, which begins after helium has begun fusing steadily in the core.

## Compilation Strategy

To ensure fairness and reproducibility, both compilers were configured using systematically constructed optimisation strategies. We began from a common baseline configuration and then explored a small set of progressively more aggressive optimisation and vectorisation settings for each compiler.

### Native Architecture Optimisation

Each build of MESA was compiled on the same CPU architecture where the corresponding benchmark was run. This "build native, run native" approach allows each compiler to target the full feature set of the underlying hardware, including microarchitecture-tuned code generation and scheduling heuristics, in those configurations where native tuning is enabled.

For the Intel ifx compiler, the `-xHost` flag was used to enable optimisation for the local host architecture. For GCC, the equivalent `-march=native` flag was applied, ensuring automatic selection of architecture‑specific instruction sets and tuning parameters.

### Per-compiler Optimisation Configurations

Out of the box, MESA provides only a `compile-settings-gnu.mk` file, which defines the compilation flags for GCC/gfortran. This baseline configuration uses `-O2` with conservative floating-point settings (no fast-math). For this study, we created a corresponding `compile-settings-ifx.mk` file for the Intel ifx compiler, designed to mirror the intent of the GNU configuration as closely as possible and then extend it in parallel ways.

For both compilers we evaluated five optimisation configurations, starting from the baseline `-O2` setup and progressively enabling more aggressive code generation, vectorisation, and use of AVX‑512. These configurations were managed in a dedicated repository: each configuration corresponds to a specific commit that updates the MESA compile-settings file for the given compiler. Before each build, we checked out the appropriate commit, rebuilt MESA, and then ran the benchmark. This ensured that:

- the full set of flags for any given configuration is version-controlled and reproducible,
- both compilers are tested under closely matched levels of optimisation and vectorisation,
- changing configurations does not rely on manual edits during the benchmarking process.

The five configurations are summarised in Table X (commit hashes refer to the optimisation-settings repository [TODO: provide link]):

| Commit hash | Description                                                                                                                                                                                                                                                             |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2bc2f83     | Baseline `-O2` build targeting a generic architecture with conservative floating-point settings (no fast-math).                                                                                                                                                         |
| 443cecf     | `-O2` with native architecture tuning and, for GCC, `-ftree-vectorize` enabled to move from a "very cheap" to a "cheap" vectorisation cost model; this brings GCC's auto-vectorisation behaviour closer to Intel ifx at `-O2`, which already emits vector instructions. |
| 93c4320     | `-O2` targeting native architecture and SIMD, FMA, and fast-math options enabled.                                                                                                                                                                                       |
| 1b52195     | Upgrade to `-O3` while retaining the previous optimisations, enabling more aggressive inlining and loop transformations.                                                                                                                                                |
| e6f5155     | As above, but with AVX‑512 explicitly requested on supported architectures to study the impact of wide vector units.                                                                                                                                                    |

In the results section we compare GCC and Intel ifx across these configurations to disentangle the effects of baseline optimisation level, native tuning, fast-math, and explicit AVX‑512 usage.

## Benchmark Procedure

### Avoiding I/O Interference

To ensure that our measurements reflect the cost of the core stellar evolution calculations rather than setup overheads, each integration test was executed in two stages. First, we ran the test once with `MESA_SKIP_OPTIONAL` unset to generate all required input and auxiliary files, and discarded this initial wall-clock time. We then enabled `MESA_SKIP_OPTIONAL` and executed the test, timing only this final critical evolution phase. All reported results are based on these timed runs, so the measured wall-clock times predominantly reflect the most computationally intensive part of the workflow while avoiding contamination from one-off file-creation and setup costs.

### OpenMP Parallelism and Thread Counts

MESA employs shared-memory parallelisation via OpenMP only. No distributed (MPI) parallelism is used. Each compiler/architecture combination was benchmarked using the following OpenMP thread counts:

- 8 threads
- 16 threads
- 32 threads
- 64 threads

These values span the typical high-performance operating regime for MESA on modern many-core nodes while remaining within the per-node core counts of the systems under test.

### Repeated Trials and Time Selection

To reduce the effect of transient system noise (e.g. filesystem variability), each test and thread-count combination was executed three times under otherwise identical conditions. For each data point:

- we report the minimum wall-clock time observed across the three runs,
- in practice, the three measurements for a given configuration were found to be very close to each other, suggesting limited performance variability on the dedicated benchmark nodes.

This "best-of-three" strategy mitigates the impact of occasional outliers due to background system activity and provides an estimate of the best achievable performance under typical conditions, rather than an average over all runs.

# Results

In this section we present the performance of GCC (gfortran) and Intel ifx across the four MESA integration tests, four Intel CPU architectures, and multiple OpenMP thread counts. For each combination of test case, architecture, compiler, and thread count, we report the minimum wall-clock time from three repeated runs, as described in the methodology.

All results correspond to native builds compiled on the same architecture where they were executed, using the common MESA configuration and compiler flags described earlier.
