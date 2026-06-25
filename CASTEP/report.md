# Introduction

High-performance scientific computing relies heavily on the ability of software to efficiently utilise modern CPU architectures.
As processor generations evolve — introducing wider vector units, expanded core counts, heterogeneous memory hierarchies, and new instruction sets — the performance of large scientific applications is increasingly determined by compiler quality and the effectiveness of architecture-specific optimisations.
Benchmarking compilers across multiple CPU families is therefore essential for understanding how scientific workloads behave on modern HPC systems and for guiding users toward best practices.

CASTEP is a software package to calculate the properties of materials. It is based on quantum mechanics, in a form known as density functional theory, and can simulate a wide range of materials proprieties including energetics, structure at the atomic level, vibrational properties, and many experimental characterisation methods, such as infra-red and Raman spectra, NMR, and core-level spectra.
The CASTEP tool is frequently used by researchers, and when it it requires HPC scale resources. This makes it an applicable and realistic case study to benchmark the impact of different architectures and compilers.
When being used on the BlueBEAR HPC in Birmingham the CASTEP executable is normally loaded as a pre-prepared module. Instead we will be building it from source throughout this study.
CASTEP can be used for multiple types of computation, we treated 5 different types of computation as independent case studies.

This work investigates the performance of GCC (gfortran) and the Intel Fortran Compiler (ifx) across several CPU architectures available on the University of Birmingham's BlueBEAR HPC cluster — Ice Lake, Sapphire Rapids, and Emerald Rapids — as well as on Lenovo's LENOX system with Granite Rapids processors.
Particular attention was paid to using native builds on each target CPU, these setting give each compiler full access to platform-specific features such as AVX‑512 variants and tuning heuristics.

The goal of this study is to present a transparent, reproducible, and architecture-aware comparison of compiler performance for MESA across a diverse range of current Intel microarchitectures.
The results aim to guide HPC users, system administrators, and researchers in choosing optimal compiler configurations for scientific workflows on current x86-based platforms.


# Methodology

## Benchmark Platforms

Performance tests were conducted on four Intel CPU architectures:

- **Ice Lake** (BlueBEAR): dual-socket nodes with 2 × 36-core Intel Xeon Platinum 8360Y CPUs (72 cores total).
- **Sapphire Rapids** (BlueBEAR): dual-socket nodes with 2 × 56-core Intel Xeon Platinum 8480CL CPUs (112 cores total).
- **Emerald Rapids** (BlueBEAR): dual-socket nodes with 2 × 56-core Intel Xeon Platinum 8570 CPUs (112 cores total).
- **Granite Rapids** (LENOX): dual-socket nodes with 2 × 128-core Intel Xeon 6 6980P CPUs (256 cores total).

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
- Convergence calculations
- Dispersion calculations
- Electronic calculations
- Phonon mode calculation
- Geometry Optimisation

The following descriptions are quoted directly from the [CASTEP documentation](https://castep-docs.github.io/castep-docs/).

### Single Point (Convergence & Dispersion)

The CASTEP Energy task allows you to calculate the total energy of the specified 3D periodic system, as well its physical properties.

In addition to the total energy, the forces on atoms are reported at the end of the calculation. A charge density file is also created, allowing you to visualize the spatial distribution of the charge density using the Materials Visualizer. The electronic energies at the Monkhorst-Pack k-points used in the calculation are also reported, so that you can generate a density of states chart during CASTEP analysis.

The Energy task is useful for studying the electronic properties of systems for which reliable structural information is available. It can also be used to calculate an equation of state (that is, a pressure-volume and/or energy-volume dependence) for high-symmetry systems with no internal degrees of freedom, as long as the Stress property is specified.


### Spectral (Electronic)

As well as computing band-structures and densities of states CASTEP has several tools for analysis of the electronic structure including:
+ Mulliken population analysis
+ Hirshfeld population analysis
+ Electron Localisation Functions (ELF)

CASTEP employs several electronic solvers. The default solver uses a density mixing (DM) algorithm in which the Kohn-Sham equations are solved for a fixed input density, and then a separate density mixing algorithm is used to evolve the density towards the groundstate.

For difficult to converge systems Ensemble Density Functional Theory (EDFT) can be used; this method is extremely robust, but much more computationally demanding than density mixing methods.

### Phonon

CASTEP can compute vibrational (phonon) modes for metals and insulators using either of density functional perturbation theory (DFPT) or finite displacements in conjunction with supercells. In addition to the traditional method of a user-specified supercell (a.k.a. the "direct method") CASTEP implements a new method which automatically selects and generates a series of supercells commensurate with the desired phonon wavevector criteria.

### Geometry Optimisation

The CASTEP Geometry Optimization task allows you to refine the geometry of a 3D periodic system to obtain a stable structure or polymorph. This is done by performing an iterative process in which the coordinates of the atoms and possibly the cell parameters are adjusted so that the total energy of the structure is minimized.

CASTEP geometry optimization is based on reducing the magnitude of calculated forces and stresses until they become smaller than defined convergence tolerances. It is also possible to specify an external stress tensor to model the behavior of the system under tension, compression, shear, and so on. In these cases the internal stress tensor is iterated until it becomes equal to the applied external stress.

The process of geometry optimization generally results in a model structure that closely resembles the real structure.

## Sample crystal lattice definitions

The cell file of CASTEP contains all of the information about the crystal lattice and the atomic position.
A sample of cell files were provided by Andrew Morris & Mario Ongkiko.
They include structures for GaAs (semiconductor, 8 atoms), LiFePO4 (battery material, 24 atoms), CaTiO3 (perovskite, 40 atoms), and MOF-5 (porous framework, 424 atoms). 


# Castep Compilation & Optimisation

CASTEP ships with configuration scripts for both gfortan and Intel "out of the box".
The Intel version was hard coded to use the older *ifort*, rather than the newer *ifx*, so we needed to change that manually.

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

We investigated the existence of I/O using VTune and found that there was very little file I/O. The use of MPI, however, introduces a significant amount of network I/O. Running CASTEP without MPI could reduce this, however, this is not how it is used in practice on BlueBEAR, so we did not persue that route as it would not reflect real use-case runtimes. 

We added the slurm flags `--nodes=1` and `--exclusive` so that all the tasks were performed on the same node, and that node was not being used simultaneously by other jobs.
This could reduce the variability of impact of intercommunication network I/O between MPI tasks, but we didn't eliminate it altogether to avoid a case where our findings diverge from any experience of a CASTEP user.

### Node allocation, MPI Parallelism, and Thread Counts

CASTEP employs distributed (MPI) parallelism.
CASTEP permits the use of shared-memory parallelisation via OpenMP, however we found allocating all core to MPI tasks led to consistently faster runtimes. 
We therefore set OpenMP threads to 1 throughout.
Each compiler/architecture combination was benchmarked using the following MPI task counts:

- 16 tasks
- 24 tasks
- 32 tasks
- 48 tasks
- 64 tasks

These values cover typical HPC core counts while remaining on a single node. We used the node exclusively throughout the simulation, even if not all the cores of the node were utilised. This was to ensure other simultaneous use of the node did not negatively impact the runtimes.

### Repeated Trials and Time Selection

Upon successful completion, CASTEP reports the total time taken for the simulation. We retrieved this value from the output files for each test case and used it as the metric of runtime.
We believed this to be a better metric of time taken than using the wall time of the process. Using CASTEP's reported time will avoid including process startup time, and any overhead such as setting up and taking down the MPI tasks, thus focusing only on the time taken which can be dependent on the compiler.

Since each simulation type was performed using the sample of crystal lattice definitions, the times for the multiple definitions were summed to give a time for the simulation type. This implicitly gives higher weighting to longer running simulations.

To reduce the effect of transient system noise (e.g. inter-task networking), each test and number of MPI tasks combination was executed three times under otherwise identical conditions.
For each data point:
This "best-of-three" strategy provides an estimate of the best achievable performance under typical conditions, rather than an average over all runs. We are confident taking this approach, given the absense of outliers across the three runs in all cases.

# Results

In this section we present independent plots of the five CASTEP study type. Each plots is further faceted by the four Intel CPU architectures.
On each plot six different curves are plotted, each corresponding to a particular choice of compiler and compiler settings.
Each curve shows the lowest reported runtime from three repeated runs, plotted against a range of MPI tasks.

## Plots

Plotting the results of all 
+ 5 tests,
+ each with 2 compilers,
+ each with 3 optimisation levels, 
+ across a axis of MPI task numbers.

<table>
  <tr>
    <img src="plots/timing_Convergence.png" alt="timing_Convergence">
  </tr>
  <tr>
    <img src="plots/timing_Dispersion.png" alt="timing_Dispersion">
  </tr>
  <tr>
    <img src="plots/timing_Electronic.png" alt="timing_Electronic">
  </tr>
    <img src="plots/timing_Geometry.png" alt="timing_Geometry">
  </tr>
  <tr>
    <img src="plots/timing_Phonon.png" alt="timing_Phonon">
  </tr>
</table>

## Summaries 


### Impact of increased MPI tasks

Increasing the number of MPI tasks improved the runtime in almost all cases, although never approached linear scaling.
This poor scaling is probably linked the earlier VTune investigation which found that there is a high overhead of network IO when CASTEP uses MPI.
The scaling on the Ice Lake was poorest, Rapid architectures were better, but still far below linear. 


| CPU Architecture| Compiler | Average speed increase going from 16 to 64 Tasks |
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


The best MPI parallelisation is achieved on the Granite Rapids using ifx.
On all other architectures, MPI parallelisation is worse and the GNU toolchain carries it out better than the Intel toolchain.


### Impact of compiler


Between the 5 test cases, 3 optimisation levels, 4 CPU architectures, and 5 task-counts, there were 300 combinations tested.
Of these, ifx performed better in 255 (85%) of cases. 
ifx outperformed gfortran by a ratio of  1.1709 in time (gfortran took 1.1709x as much time as ifx, taking the mean across all test combinations).
In the majority of cases, ifx outperformed gfortran.


| ifx faster | ifx faster % | avg ratio |
| -------- | -------- | --------- |
| 255/300  | 85%      | 1.1709    |

The improvement varied by the different simulations, numbers of task, and CPU architectures.

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


| CPU architecture   | ifx faster | ifx faster % | avg ratio |
| ------------------ | -------- | -------- | --------- |
| Ice Lake           | 55/75    | 73%      | 1.0558    |
| Sapphire Rapids    | 61/75    | 81%      | 1.0588    |
| Emerald Rapids     | 68/75    | 91%      | 1.0796    |
| Granite Rapids     | 71/75    | 95%      | 1.4891    |



ifx excelled on the Granite Rapids, where it performed better in 71/75 (95%) of cases and an average ratio of 1.4891 times the gfortran speed.
ifx is least strong on The Ice Lakes, though it was still faster in 55/75 (73%) cases, albeit with an average ratio of only 1.0558, indicating far closer times.

ifx performed strongest with between 24-48 MPI tasks, with  the lower count of 16 having closer average ratio, and the higher count of 64 having fewer won cases.