# Introduction

High-performance scientific computing relies heavily on the ability of software to efficiently utilise modern CPU architectures.
As processor generations evolve-introducing wider vector units, expanded core counts, heterogeneous memory hierarchies, and new instruction sets-the performance of large scientific applications is increasingly determined by compiler quality and the effectiveness of architecture-specific optimisations.
Benchmarking compilers across multiple CPU families is therefore essential for understanding how scientific workloads behave on modern HPC systems and for guiding users toward best practices.

The Modules for Experiments in Stellar Astrophysics (MESA) code represents a demanding and widely used benchmark for such studies.
Written primarily in Fortran, MESA performs computationally intensive stellar evolution simulations with substantial floating-point workloads and OpenMP-based node-level parallelism.
By combining intensive mathematical computations with high-performance data handling, it serves as an ideal benchmark for evaluating compilers on real-world scientific workloads.

Stars can exist and evolve in a number of distinctive states and MESA aims to be able to simulate as many of them as are feasible.
During these different states, the simulations will vary in a number of computationally significant ways, including the appropriate resolution in time and space and how the simulation is expected to scale with increasing numbers of CPU cores.
We experimented using a number of different integration tests that represent different kinds of stars.
Though this particular selection is unlikely to have exercised the full range of MESA's capabilities, the cases were selected to include some of the most computationally intensive.

This work investigates the performance of GCC (gfortran) and the Intel Fortran Compiler (ifx) across several CPU architectures available on the University of Birmingham's BlueBEAR HPC cluster-Ice Lake, Sapphire Rapids, and Emerald Rapids-as well as on Lenovo's LENOX system with Granite Rapids processors.
Because MESA does not provide Intel-specific Makefile options by default, a custom build configuration was developed to ensure that both compilers are tested with fair and equivalent optimisation strategies.
Particular attention was paid to using native builds on each target CPU, ensuring each compiler has full access to platform-specific features such as AVX‑512 variants and tuning heuristics.

The goal of this study is to present a transparent, reproducible, and architecture‑aware comparison of compiler performance for MESA across a diverse range of current Intel microarchitectures.
The results aim to guide HPC users, system administrators, and researchers in choosing optimal compiler configurations for scientific workflows on current x86-based platforms.

# Methodology

## Benchmark Platforms

Performance tests were conducted on four Intel CPU architectures:

- **Ice Lake** (BlueBEAR): dual-socket nodes with 2 × 36-core Intel Xeon Platinum 8360Y CPUs (72 cores total).
- **Sapphire Rapids** (BlueBEAR): dual-socket nodes with 2 × 56-core Intel Xeon Platinum 8480CL CPUs (112 cores total).
- **Emerald Rapids** (BlueBEAR): dual-socket nodes with 2 × 56-core Intel Xeon Platinum 8570 CPUs (112 cores total).
- **Granite Rapids** (LENOX): dual-socket nodes with 2 × 128-core Intel Xeon 6 6980P CPUs (256 cores total).

MESA is parallelised using shared-memory OpenMP only; no distributed-memory (MPI) parallelism is employed.
All benchmarks were therefore run on a single compute node per job.

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

All MESA builds used a common configuration intended to reflect a typical production setup while keeping the benchmark focused on the core stellar evolution kernels.
The following options were used:

```
PROFILE=release
WITH_OPENMP=yes
WITH_ADIPLS=no
WITH_CRLIBM=no
WITH_GYRE=no
WITH_PGSTAR=no
```

MESA itself is written primarily in Fortran and uses OpenMP for intra-node parallelism.
Because MESA does not include Intel-targeted build rules by default, a new Makefile configuration was created for the Intel ifx compiler.
This ensured:

- correct handling of Intel Fortran syntax and modules,
- consistent linking behaviour across compilers,
- parity between GCC and Intel build options (including OpenMP and optimisation flags).

We are working with the MESA maintainers to make this configuration available upstream for potential inclusion in the MESA mainline.

### Overview of MESA Integration Tests

Longer MESA integration tests are designed so that the final, critical phase of the evolution can be run in isolation from the the rest of the test, which is regarded as only being necessary to prepare the input for the actual test.
Whether a test runner includes the whole simulation or just the final critical part is controlled by the parameter `MESA_SKIP_OPTIONAL`.
If this parameter is set (to any value), only the final part of the simulation is executed.
To focus on the most computationally demanding phases and keep overall runtimes manageable, all benchmarks in this study were run with `MESA_SKIP_OPTIONAL` enabled, thereby executing only the critical final stage of each integration test.

To keep the runtime of the whole test suite reasonable, MESA's integration tests are generally configured at lower spatial and temporal resolutions than researchers would use for full scientific studies.
However, the relative computational intensity between different phases and test cases is maintained, making them suitable for comparative performance analysis.

### Selected Benchmark Cases

We selected four MESA integration tests that collectively span a range of stellar types and evolutionary phases, with an emphasis on computationally challenging regimes:

1. `wd_stable_h_burn`
2. `20M_pre_ms_to_core_collapse`
3. `15M_dynamo`
4. `5M_cepheid_blue_loop`

Each of these tests corresponds to a physically distinct scenario and stresses MESA in different ways, as described below.

#### wd_stable_h_burn

White dwarfs are very dense stars that are the remnant cores of stars up to about eight times as massive as the Sun.
They sometimes exist near enough to companion stars that some of the companion (which is still mostly hydrogen) steadily settles on the white dwarf's surface and it can begin to shine by fusing the hydrogen into helium.
This test case approximates such a situation with parameters that are finely tuned to produce steady fusion of hydrogen into helium.
The extreme density of the white dwarf and thinness of the hydrogen layer make this simulation computationally demanding.

#### 20M_pre_ms_to_core_collapse

This integration test simulates the life of a star 20 times as massive as the Sun from its birth to the moments before it explodes as a supernova (which MESA itself does not simulate).
Pre-supernova evolution is generally one of the most computationally intensive phases, as it requires a high-resolution mesh with many chemical species under extreme physical conditions.
For this benchmark we used only the final stage of this otherwise very long integration test, which follows the star from when silicon begins to fuse into even heavier elements (around 4 billion kelvin) until the central parts of the star have begun to collapse.

#### 15M_dynamo

MESA is able to approximately include changes in a star's structure and evolution caused by rotation.
This test case serves to check that the additional computations required in this mode are functioning correctly by simulating a star 15 times as massive as the Sun from birth until about halfway through a late stage in which helium fuses into carbon and oxygen in the core.

#### 5M_cepheid_blue_loop

This integration test follows a star 5 times as massive as the Sun as it fuses both helium in the core and hydrogen in a thin shell around the helium core.
The increasing power from the core interacts with the hydrogen shell and is known to require enough spatial resolution around the shell to correctly follow its behaviour.
Our benchmarking is only based on the second part of the test, which begins after helium has begun fusing steadily in the core.

## Compilation Strategy

To ensure fairness and reproducibility, both compilers were configured using systematically constructed optimisation strategies.
We began from a common baseline configuration and then explored a small set of progressively more aggressive optimisation and vectorisation settings for each compiler.

### Native Architecture Optimisation

Each build of MESA was compiled on the same CPU architecture where the corresponding benchmark was run.
This "build native, run native" approach allows each compiler to target the full feature set of the underlying hardware, including microarchitecture-tuned code generation and scheduling heuristics, in those configurations where native tuning is enabled.

For the Intel ifx compiler, the `-xHost` flag was used to enable optimisation for the local host architecture.
For GCC, the equivalent `-march=native` flag was applied, ensuring automatic selection of architecture‑specific instruction sets and tuning parameters.

### Per-compiler Optimisation Configurations

Out of the box, MESA provides only a `compile-settings-gnu.mk` file, which defines the compilation flags for GCC/gfortran.
This baseline configuration uses `-O2` with conservative floating-point settings (no fast-math).
For this study, we created a corresponding `compile-settings-ifx.mk` file for the Intel ifx compiler, designed to mirror the intent of the GNU configuration as closely as possible and then extend it in parallel ways.

For both compilers we evaluated five optimisation configurations, starting from the baseline `-O2` setup and progressively enabling more aggressive code generation, vectorisation, and use of AVX‑512.
These configurations were managed in a dedicated repository: each configuration corresponds to a specific commit that updates the MESA compile-settings file for the given compiler.
Before each build, we checked out the appropriate commit, rebuilt MESA, and then ran the benchmark.
This ensured that:

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

To ensure that our measurements reflect the cost of the core stellar evolution calculations rather than setup overheads, each integration test was executed in two stages.
First, we ran the test once with `MESA_SKIP_OPTIONAL` unset to generate all required input and auxiliary files, and discarded this initial wall-clock time.
We then enabled `MESA_SKIP_OPTIONAL` and executed the test, timing only this final critical evolution phase.
All reported results are based on these timed runs, so the measured wall-clock times predominantly reflect the most computationally intensive part of the workflow while avoiding contamination from one-off file-creation and setup costs.

### OpenMP Parallelism and Thread Counts

MESA employs shared-memory parallelisation via OpenMP only.
No distributed (MPI) parallelism is used.
Each compiler/architecture combination was benchmarked using the following OpenMP thread counts:

- 8 threads
- 16 threads
- 32 threads
- 64 threads

These values span the typical high-performance operating regime for MESA on modern many-core nodes while remaining within the per-node core counts of the systems under test.

### Repeated Trials and Time Selection

To reduce the effect of transient system noise (e.g. filesystem variability), each test and thread-count combination was executed three times under otherwise identical conditions.
For each data point:

- we report the minimum wall-clock time observed across the three runs,
- in practice, the three measurements for a given configuration were found to be very close to each other, suggesting limited performance variability on the dedicated benchmark nodes.

This "best-of-three" strategy mitigates the impact of occasional outliers due to background system activity and provides an estimate of the best achievable performance under typical conditions, rather than an average over all runs.

# Results

In this section we present the performance of GCC (gfortran) and Intel ifx across the four MESA integration tests, four Intel CPU architectures, and multiple OpenMP thread counts.
For each combination of test case, architecture, compiler, and thread count, we report the minimum wall-clock time from three repeated runs, as described in the methodology.

All results correspond to native builds compiled on the same architecture where they were executed, using the common MESA configuration and compiler flags described earlier.
The discussion focuses primarily on the 16- and 32-thread configurations, which represent the practical sweet spot for these benchmarks; at 64 threads the scaling curves have typically flattened, and the relative ordering between configurations can become less representative of the regime most users would employ.

Across all architectures and tests, we observe broadly similar qualitative scaling behaviour.
For both compilers, increasing the number of OpenMP threads from 8 to 64 reduces the wall-clock time substantially, but the speedups are consistently sub-linear.
The step from 8 to 16 threads captures the largest relative gain, with diminishing returns at each subsequent doubling.
The fastest configurations achieve speedups ranging from roughly 2x for tests with high serial fractions up to approximately 5x for the most parallelisable case (`20M_pre_ms_to_core_collapse`), rather than the ideal factor of eight that would be expected from perfect scaling.

The degree of parallel speedup also depends noticeably on the integration test.
The APS profiles collected on Granite Rapids show that the tests differ substantially in their serial time fraction, which directly limits scalability. `20M_pre_ms_to_core_collapse` has the lowest serial fraction (approximately 10% of elapsed time at 64 threads) and achieves the best scaling, with speedups of approximately 5x from 8 to 64 threads.
At the other extreme, `5M_cepheid_blue_loop` has the highest serial fraction (approximately 60% of elapsed time), and its scaling stalls at roughly 2x. `15M_dynamo` also has a high serial fraction (approximately 45-50% of elapsed time), yielding a speedup of roughly 2.5x. `wd_stable_h_burn` falls in between, with a moderate serial fraction (approximately 30% of elapsed time) and a speedup of roughly 4x.
This clean monotonic relationship — higher serial fraction directly predicts worse scaling — indicates that the tests with the lowest serial fractions are able to exploit additional threads most effectively, while those dominated by serial or synchronisation-heavy regions see diminishing returns beyond 16-32 threads.

Compiler choice affects the absolute runtimes but does not fundamentally change these trends.
Across all 320 configurations (4 architectures x 4 tests x 5 methods x 4 thread counts), ifx is faster than gfortran in approximately 72% of cases, with an average speed advantage of roughly 24%.
However, the margin is highly test-dependent. On `5M_cepheid_blue_loop`, ifx wins every configuration by a wide margin (73% faster on average); even the most aggressively optimised gfortran build (Method 5) is slower than the baseline ifx build (Method 1).
On `20M_pre_ms_to_core_collapse`, ifx wins 84% of configurations. On `15M_dynamo`, ifx wins 71% of configurations but the average advantage is only about 6%, and at higher thread counts gfortran often matches or slightly exceeds ifx.
On `wd_stable_h_burn`, gfortran wins 65% of configurations, particularly at higher thread counts, though ifx becomes more competitive at lower thread counts and with more aggressive optimisation flags.
Overall, both compilers are capable of exploiting the available hardware parallelism, but ifx extracts somewhat more per-core performance on average, with the notable exception of `wd_stable_h_burn` where gfortran holds an edge.

The five optimisation configurations, which move from a conservative `-O2` baseline to `-O3` with fast-math and explicit AVX-512, have uneven and sometimes counter-intuitive impact across tests, architectures, and compilers.
For ifx, moving beyond the baseline Method 1 generally improves performance, with the largest gains concentrated at the step from Method 1 to Method 2 or 3 (native tuning and fast-math).
Further escalation to Method 4 (`-O3`) or Method 5 (explicit AVX-512) yields diminishing or no additional returns for most test-architecture combinations.
For gfortran, the picture is more mixed: on some architectures and tests the more aggressive methods consistently improve performance, while on others they produce regressions.
The most striking example is `20M_pre_ms_to_core_collapse` with gfortran on Granite Rapids, where enabling AVX-512 (Method 5) makes performance approximately 20% worse than the baseline `-O2` build, suggesting that the auto-vectorised code for this test's kernels does not benefit from wider vectors and may suffer from increased register pressure or less favourable instruction scheduling.

The test most sensitive to compiler optimisation choices is `5M_cepheid_blue_loop`.
For both compilers and across all architectures, moving from Method 1 to any of Methods 2-5 produces large and consistent speedups (typically 40-90% for gfortran, 15-25% for ifx).
This test also shows the largest compiler gap: ifx outperforms gfortran by factors of 1.5-3x depending on configuration.
The combination of high sensitivity to optimisation and large compiler differences suggests that `5M_cepheid_blue_loop` contains kernels that are particularly well-suited to the vectorisation and loop transformation capabilities of modern compilers, and that ifx handles these kernels substantially more effectively than gfortran.

For `20M_pre_ms_to_core_collapse` with ifx, Methods 3, 4, and 5 yield broadly similar performance at 16 and 32 threads, with no single method consistently winning across all architectures.
This indicates that once fast-math and native tuning are enabled (Method 3), the additional benefits from `-O3` and explicit AVX-512 are limited for this test.
With gfortran on this test, the picture is more erratic: on Granite Rapids, Methods 3 and 5 are slower than Method 1 while Method 4 shows a modest improvement, and on other architectures the results vary with no clear trend.

For `15M_dynamo`, ifx wins at 8-32 threads across all architectures, though the margin over gfortran is most pronounced at lower thread counts (up to 25-30% on Emerald and Granite Rapids at 8 threads) and narrows considerably at 32 threads.
The benefit of moving beyond Method 1 is modest for gfortran (typically up to 10%) and more variable for ifx (typically 10-25%).
An interesting anomaly is that on Ice Lake, Sapphire Rapids, and Emerald Rapids, the baseline ifx build (Method 1) is slower than every gfortran configuration at 8-32 threads, while on Granite Rapids the baseline ifx build outperforms all gfortran configurations at 8 and 16 threads — possibly reflecting the different compiler versions used on the LENOX cluster (gfortran 14.2.0, ifx 2025.2.1) compared to BlueBEAR (gfortran 13.3.0, ifx 2024.2.0).

For `wd_stable_h_burn`, the compiler comparison flips depending on architecture.
On Ice Lake, Sapphire Rapids, and Emerald Rapids, ifx at Methods 3-5 is competitive with or slightly faster than gfortran at 16 threads, though gfortran tends to regain the advantage at 32 threads.
On Granite Rapids, gfortran wins across all methods and thread counts, typically by 5-15%.
This architectural dependence may again be influenced by the different compiler versions on the two clusters.
The overall performance differences on this test are modest — the best configurations across both compilers are within 10-20% of each other — suggesting that `wd_stable_h_burn` is not strongly sensitive to compiler choice or optimisation flags.

The APS profiles collected on Granite Rapids help to explain the scaling behaviour and provide a cross-check that the timing results are consistent with underlying microarchitectural metrics.
For ifx, APS reports substantial serial time fractions that increase with thread count — ranging from approximately 5% at 16 threads to 11% at 64 threads for the most parallelisable test (`20M_pre_ms_to_core_collapse`), and from approximately 35-45% at 16 threads to 45-65% at 64 threads for the more serial-dominated tests (`15M_dynamo` and `5M_cepheid_blue_loop`).
APS also reports non-negligible OpenMP imbalance, typically 5-10% at 16 threads growing to 10-20% at 64 threads.
Together, these factors naturally lead to sub-linear speedups as thread count increases, since the serial and imbalanced components dominate an increasing fraction of the total wall-clock time.
Note that APS was unable to report serial time and OpenMP imbalance for gfortran builds, as these metrics rely on the Intel OpenMP runtime's instrumentation interface, which is not available when using the GNU OpenMP runtime.

The APS metrics also reveal differences between tests that help explain their contrasting responses to compiler optimisation.
For ifx, `5M_cepheid_blue_loop` stands out with the highest vectorisation percentage (approximately 50% on Method 1) and the lowest cache stall fraction (approximately 10%), consistent with its strong response to compiler optimisation.
By contrast, `20M_pre_ms_to_core_collapse` shows moderate vectorisation (29%) and low cache stalls (12%) on Method 1, but moving to Method 5 actually reduces vectorisation (to 18%) and increases cache stalls (to 21%), consistent with the flat or negative returns from aggressive optimisation observed for this test.
For `15M_dynamo`, ifx shows high vectorisation on Method 1 (42%) but also the highest serial time fraction (approximately 45-50% of elapsed time), which limits the impact of any per-kernel optimisation gains on total runtime.
For `wd_stable_h_burn`, ifx shows moderate vectorisation (32%) and low cache stalls (10%) on Method 1, with modest improvements from more aggressive methods.

For gfortran, the APS profiles show much lower vectorisation percentages across all tests (2-15% depending on method, compared to 17-51% for ifx) and typically higher cache stall fractions (roughly 12-31% vs 4-24% for ifx).
This difference in vectorisation capability is a likely contributor to ifx's overall performance advantage, particularly on tests like `5M_cepheid_blue_loop` where vectorisation-friendly kernels dominate.
The gfortran profiles also show that moving from the baseline Method 1, which uses a very conservative vectorisation cost model, to Method 4 substantially increases vectorisation on `5M_cepheid_blue_loop` (from 2% to 15%), while the effect is more modest on the other three tests (from roughly 3% to 5-11%), mirroring the benchmark timing patterns where `5M_cepheid_blue_loop` benefits most from aggressive optimisation under gfortran.

The benchmark timing curves are therefore consistent with the profiling data: tests and compiler configurations that APS identifies as more vectorisation-friendly and less stall-prone (`5M_cepheid_blue_loop` with ifx) exhibit the largest benefits from aggressive optimisation and from choosing ifx over gfortran, while tests with high serial fractions (`15M_dynamo`) or limited vectorisation gains (`20M_pre_ms_to_core_collapse` with gfortran) show more modest and sometimes irregular speedups.
This coherence between timing and profiling results strengthens the overall conclusion that differences in per-test behaviour and compiler gains can be traced back to a small number of underlying hardware and algorithmic factors rather than to noise or measurement artefacts.
