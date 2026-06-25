# Introduction

High-performance scientific computing relies heavily on the ability of software to efficiently utilise modern CPU architectures.
As processor generations evolve — introducing wider vector units, expanded core counts, heterogeneous memory hierarchies, and new instruction sets — the performance of large scientific applications is increasingly determined by compiler quality and the effectiveness of architecture-specific optimisations.
Benchmarking compilers across multiple CPU families is therefore essential for understanding how scientific workloads behave on modern HPC systems and for guiding users toward best practices.

CASTEP is a software package to calculate the properties of materials ([website](https://www.castep.org), [documentation](https://castep-docs.github.io/castep-docs/)). It is based on quantum mechanics, in a form known as density functional theory, and can simulate a wide range of materials proprieties including energetics, structure at the atomic level, vibrational properties, and many experimental characterisation methods, such as infra-red and Raman spectra, NMR, and core-level spectra.
The CASTEP tool is frequently used by researchers, and this requires HPC scale resources. This makes it an applicable and realistic case study to benchmark the impact of different architectures and compilers.
CASTEP can be used for multiple types of computation, we treated 5 different types of computation as independent case studies.

This work investigates the performance of GCC (gfortran) and the Intel Fortran Compiler (ifx) across several CPU architectures available on the University of Birmingham's BlueBEAR HPC cluster — Ice Lake, Sapphire Rapids, and Emerald Rapids — as well as on Lenovo's LENOX system with Granite Rapids processors.
Particular attention was paid to using native builds on each target CPU, these setting give each compiler full access to platform-specific features such as AVX‑512 variants and tuning heuristics.

The goal of this study is to present a transparent, reproducible, and architecture-aware comparison of compiler performance for MESA across a diverse range of current Intel microarchitectures.
The results aim to guide HPC users, system administrators, and researchers in choosing optimal compiler configurations for scientific workflows on current x86-based platforms.

# Methodology

## Benchmark Platforms

Performance tests were conducted on four Intel CPU architectures:

+ **Ice Lake** (BlueBEAR): dual-socket nodes with 2 × 36-core Intel Xeon Platinum 8360Y CPUs (72 cores total).
+ **Sapphire Rapids** (BlueBEAR): dual-socket nodes with 2 × 56-core Intel Xeon Platinum 8480CL CPUs (112 cores total).
+ **Emerald Rapids** (BlueBEAR): dual-socket nodes with 2 × 56-core Intel Xeon Platinum 8570 CPUs (112 cores total).
+ **Granite Rapids** (LENOX): dual-socket nodes with 2 × 128-core Intel Xeon 6 6980P CPUs (256 cores total).

CASTEP is parallelised using distributed-memory (MPI) parallelism. All benchmarks were therefore run on a single compute node per job.

## Compilation Platforms

We compiled the code using two different toolchains, 

The **gfortran** toolchain comprising

+ gcc
+ gfortran
+ openMPI
+ openBlas
+ fftw3

and an **ifx** toolchain comprising

+ icx
+ ifx
+ mpiifx
+ Intel MKL blas
+ Intel MKL fft

## CASTEP case studies

The 5 case studies tested are
+ [Convergence calculations](https://www.tcm.phy.cam.ac.uk/castep/documentation/WebHelp/content/modules/castep/tskcastepenergy.htm)
+ [Dispersion calculations](https://www.tcm.phy.cam.ac.uk/castep/documentation/WebHelp/content/modules/castep/tskcastepenergy.htm)
+ [Electronic calculations](https://www.castep.org/features/capabilities/electronic-properties)
+ [Phonon mode calculation](https://www.castep.org/features/capabilities/vibrational-spectroscopy)
+ [Geometry Optimisation](https://www.tcm.phy.cam.ac.uk/castep/documentation/WebHelp/content/modules/castep/tskcastepgeometry.htm)

Details of each can be found via the links above to CASTEP documentation.

## Sample crystal lattice definitions

The cell file of CASTEP contains all of the information about the crystal lattice and the atomic position.
A sample of cell files were provided by Professor Andrew Morris & Dr Mario Ongkiko.
They include structures for GaAs (semiconductor, 8 atoms), LiFePO4 (battery material, 24 atoms), CaTiO3 (perovskite, 40 atoms), and MOF-5 (porous framework, 424 atoms). 

# Castep Compilation & Optimisation

CASTEP ships with configuration scripts for both gfortan and Intel "out of the box".
The shipped Intel scipt uses the older ifort compiler, rather than the newer ifx, so we needed to adjust the script to use ifx for our study.

In both cases the default optimisiation setting was `-O3`, which is already an aggressive optimisation level.
Unfortunatly `-O3` is a macro rather than a defined C++ term, so it is permitted to have different meanings to the two compilers.
Crucially `-O3` in ifx means that fast maths is applied, whereas `-O3` in gfortran still uses precise maths.
We therefore needed to adjust the default ifx compiler settings to the combination `-O3 -fp-model=precise` as our base case for ifx to make the baselines consistent. 

### Native Architecture Optimisation

Each build of CASTEP was compiled on the same CPU architecture where the corresponding benchmark was run.
This "build native, run native" approach allows each compiler to target the full feature set of the underlying hardware, including microarchitecture-tuned code generation and scheduling heuristics, in those configurations where native tuning is enabled.

For the Intel ifx compiler, the `-xHost` flag was used to enable optimisation for the local host architecture.
For GCC, the equivalent `-march=native` flag was applied, ensuring automatic selection of architecture-specific instruction sets and tuning parameters.

### Mathematical optimisations

We also test the impact of the use of precise maths or fast maths, making use of the compilation flags `-fp-model=fast` and `-mfpmath=sse`. 

These setting lead us to 3 cases with increasingly aggressive optimisation settings:

+ Base case (`-O3`, with portability and precise maths)
+ Host specific case (`-O3`, targeted to the host machine and precise maths)
+ Host specific with fast maths (`-O3`, targeted to the host machine and fast maths)

## Benchmark Procedure

### Avoiding I/O Interference

We investigated the prevalence of I/O using VTune and found that there was very little file I/O. The use of MPI, however, introduces a significant amount of network I/O. Running CASTEP without MPI could reduce this, but this is not how it is used in practice on BlueBEAR, so we did not pursue that route as it would not reflect realistic use-case runtimes. 

We added the slurm flags `--nodes=1` and `--exclusive` so that all the tasks were performed on the same node, and that node was not being used simultaneously by other jobs.
This could reduce the variability of the impact of intercommunication network I/O between MPI tasks, but we didn't eliminate it altogether to avoid a case where our findings diverge from typical CASTEP user experiences.

### Node allocation, MPI Parallelism, and Thread Counts

CASTEP employs distributed (MPI) parallelism.
CASTEP permits the use of shared-memory parallelisation via OpenMP, however we found allocating all core to MPI tasks led to consistently faster runtimes. 
We therefore set OpenMP threads to `1` throughout.
Each compiler/architecture combination was benchmarked using the following MPI task counts:

+ 16 tasks
+ 24 tasks
+ 32 tasks
+ 48 tasks
+ 64 tasks

These values cover typical HPC core counts while remaining on a single node. We used the node exclusively throughout the simulation, even if not all the cores of the node were utilised. This was to ensure other simultaneous use of the node did not negatively impact the runtimes.

### Repeated Trials and Time Selection

Upon successful completion, CASTEP reports the total time taken for the simulation. We retrieved this value from the output files for each test case and used it as the metric of runtime.
We believed this to be a better metric of time taken than using the wall time of the process. Using CASTEP's reported time will avoid including process startup time, and any overhead such as setting up and taking down the MPI tasks, thus focusing only on the time taken which can be dependent on the compiler.

Since each simulation type was performed using the sample of crystal lattice definitions, the times for the multiple definitions were summed to give a time for the simulation type. This implicitly gives higher weighting to longer running simulations.

To reduce the effect of transient system noise (e.g. inter-task networking), each test and number of MPI tasks combination was executed three times under otherwise identical conditions. We report on the fastest of these three runs.
This "best-of-three" strategy mitigates the impact of occasional outliers due to background system activity and provides an estimate of the best achievable performance under typical conditions, rather than an average over all runs.

# Results

In this section we present independent plots of the five CASTEP study types. Each plot is further faceted by the four Intel CPU architectures.
On each plot six different curves are plotted, each curve corresponding to a particular choice of compiler and compiler settings.
Each curve shows the shortest reported runtime from three repeated runs, plotted against a range of MPI tasks.

## Plots

Plotting the results of all 
+ 5 tests,
+ each with 2 compilers,
+ each with 3 optimisation levels, 
+ across an axis of MPI task numbers.

![timing_Convergence](plots/timing_Convergence.png)

![timing_Dispersion](plots/timing_Dispersion.png)

![timing_Electronic](plots/timing_Electronic.png)

![timing_Geomerty](plots/timing_Geometry.png)

![timing_Phonon](plots/timing_Phonon.png)

## Summaries 


### Impact of increased MPI tasks

Increasing the number of MPI tasks improved the runtime in almost all cases, although it never approached linear scaling.
The observed departure from linear scaling may be related to the earlier VTune investigation, which identified significant network I/O overhead when CASTEP uses MPI.
Rapid architectures showed greater performance improvements with increasing MPI task counts than Ice Lake systems, although neither approached linear scaling.


| CPU Architecture| Compiler | Average speed increase from 16 to 64 tasks |
| --------------- | -------- |  ----------- | 
| Ice Lake        | ifx      |  1.75x       |
| Sapphire Rapids | ifx      |  2.21x       |
| Emerald Rapids  | ifx      |  2.32x       |
| Granite Rapids  | ifx      |  2.43x       |
|
| Ice Lake        | gfortran |  1.76x       |
| Sapphire Rapids | gfortran |  2.31x       |
| Emerald Rapids  | gfortran |  2.53x       |
| Granite Rapids  | gfortran |  1.98x       |

Granite Rapids compiled with ifx exhibited the highest speed-up (2.43×) between 16 and 64 MPI tasks. For all other architectures, gfortran delivered comparable or greater MPI scaling than ifx, with the difference being particularly pronounced on Emerald Rapids.

### Impact of targeting the architecture

A summary of the impact of using more aggressively optimised compiler options is shown below for runs using 32 MPI tasks.
These rows have been scaled against their own `base` case, so a of less than 1.0 for either `host` or `fmath` indicates that they are less performant than the portable case. 

| CPU Architecture | Compiler | `host`|`fmath`|
|------------------|----------|-------|-------|
| Ice Lake         | ifx      | 0.938 | 1.031 |
| Sapphire Rapids  | ifx      | 0.990 | 1.072 |
| Emerald Rapids   | ifx      | 0.990 | 1.087 |
| Granite Rapids   | ifx      | 1.010 | 1.096 |
| Ice Lake         | gfortran | 1.022 | 1.025 |
| Sapphire Rapids  | gfortran | 0.963 | 0.965 |
| Emerald Rapids   | gfortran | 0.963 | 0.964 |
| Granite Rapids   | gfortran | 0.968 | 0.968 |

Targeting the native archicture improves performance in only 7 out of 16 (44%) of instances. The take away from this data is that targeting the host architecture does not offer guaranteed performance improvements over portable with `-O3` optimisation. 

### Impact of compiler

Between the 5 test cases, 3 optimisation levels, 4 CPU architectures, and 5 task-counts, there were 300 combinations tested.
Of these, ifx performed better in 255 (85%) of cases. 
Averaged across all benchmark configurations, gfortran required 1.17× the execution time of ifx (mean runtime ratio: 1.1709).
Overall, these results demonstrate a clear performance advantage for ifx across the tested workloads.


| ifx faster | ifx faster % | avg ratio |
| -------- | -------- | --------- |
| 255/300  | 85%      | 1.1709    |

The performance advantage of ifx was not uniform, varying across simulations, MPI task counts, and CPU architectures.

| CASTEP simulation  | ifx faster | ifx faster % | avg ratio |
| ------------------ | -------- | -------- | --------- |
| Convergence        | 59/60    | 98%      | 1.2143    |
| Dispersion         | 53/60    | 88%      | 1.1908    |
| Electronic         | 28/60    | 47%      | 1.0237    |
| Geometry           | 59/60    | 98%      | 1.2316    |
| Phonon             | 56/60    | 93%      | 1.1939    |


| MPI tasks          | ifx faster | ifx faster % | avg ratio |
| ------------------ | -------- | -------- | --------- |
| 16                 | 50/60    | 83%      | 1.0959    |
| 24                 | 53/60    | 88%      | 1.1379    |
| 32                 | 52/60    | 87%      | 1.2264    |
| 48                 | 53/60    | 88%      | 1.2260    |
| 64                 | 47/60    | 78%      | 1.1681    |

The performance advantage of ifx was most pronounced between 24 and 48 MPI tasks, where the average runtime ratio exceeded 1.22×. At 16 tasks, the performance difference between the compilers was smaller (1.10× on average), while at 64 tasks ifx remained faster overall but won in a smaller proportion of cases (78%).

| CPU architecture   | ifx faster | ifx faster % | avg ratio |
| ------------------ | -------- | -------- | --------- |
| Ice Lake           | 55/75    | 73%      | 1.0558    |
| Sapphire Rapids    | 61/75    | 81%      | 1.0588    |
| Emerald Rapids     | 68/75    | 91%      | 1.0796    |
| Granite Rapids     | 71/75    | 95%      | 1.4891    |

The advantage of ifx varied substantially by processor architecture. The smallest benefit was observed on Ice Lake, where ifx was faster in 55 of 75 cases (73%), with a relatively modest average runtime ratio of 1.056×.
In contrast, Granite Rapids showed the largest benefit from ifx, which outperformed gfortran in 71 of 75 cases (95%). On this architecture, gfortran required 1.489× the execution time of ifx on average, representing the largest compiler-related performance difference observed in the study.

