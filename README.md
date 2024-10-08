[![Codecov](https://codecov.io/gh/LeeoBianchi/HealpixMPI.jl/branch/main/graph/badge.svg)](https://codecov.io/gh/LeeoBianchi/HealpixMPI.jl)
[![Build Status](https://github.com/LeeoBianchi/HealpixMPI.jl/workflows/Unit%20tests/badge.svg)](https://github.com/LeeoBianchi/HealpixMPI.jl/actions/workflows/UnitTest.yml)
[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](https://leeobianchi.github.io/HealpixMPI.jl/dev/)
[![status](https://joss.theoj.org/papers/f012d3aa6c7cd644b8b9ecbe2c5a0aef/status.svg)](https://joss.theoj.org/papers/f012d3aa6c7cd644b8b9ecbe2c5a0aef)

<img src="docs/src/assets/logo.png" width="180">

# HealpixMPI.jl: an MPI-parallel implementation of the Healpix tessellation scheme in Julia

Welcome to HealpixMPI.jl, an MPI-parallel implementation of the main functionalities of [HEALPix](https://healpix.sourceforge.io/) spherical tessellation scheme, entirely coded in Julia.

This package constitutes a natural extension of the package [Healpix.jl](https://github.com/ziotom78/Healpix.jl), providing an MPI integration of its main functionalities, allowing for simultaneous shared-memory (multithreading) and distributed-memory (MPI) parallelization leading to high performance sperical harmonic transforms.

Read the full [documentation](https://leeobianchi.github.io/HealpixMPI.jl/dev) for further details.

## Installation

From the Julia REPL, run

````julia
import Pkg
Pkg.add("HealpixMPI")
````

## Usage Example

The example shows the necessary steps to set up and perform an MPI-parallel `alm2map` SHT with HealpixMPI.jl.

### Set up

We set up the necessary MPI communication and initialize Healpix.jl structures:
````julia
using MPI
using Random
using Healpix
using HealpixMPI

#MPI set-up
MPI.Init()
comm = MPI.COMM_WORLD
crank = MPI.Comm_rank(comm)
csize = MPI.Comm_size(comm)
root = 0

#initialize Healpix structures
NSIDE = 64
lmax = 3*NSIDE - 1
if crank == root
  h_map = HealpixMap{Float64, RingOrder}(NSIDE)   #empty map
  h_alm = Alm(lmax, lmax, randn(ComplexF64, numberOfAlms(lmax)))  #random alm
else
  h_map = nothing
  h_alm = nothing
end
````

### Distribution

The distributed HealpixMPI.jl data types are filled through an overload of `MPI.Scatter!`:
````julia
#initialize empty HealpixMPI structures 
d_map = DMap{RR}(comm)
d_alm = DAlm{RR}(comm)

#fill them
MPI.Scatter!(h_map, d_map)
MPI.Scatter!(h_alm, d_alm)
````

### SHT

We perform the SHT through an overload of `Healpix.alm2map` and, if needed, we `MPI.Gather!` the result in a `HealpixMap`:

````julia
alm2map!(d_alm, d_map; nthreads = 16)
MPI.Gather!(d_map, h_map)
````

This allows the user to adjust at run time the number of threads to use, typically to be set to the number of cores of your machine.

### Polarization

Since v1.0.0 HealpixMPI.jl supports polarized SHT's.
There are two different ways to distribute a `PolarizedHealpixMap` using `MPI.Scatter!`, i.e. passing one or two `DMap` output objects respectively, as shown in the following example:
````julia
MPI.Scatter!(h_map, out_d_pol_map) #here out_d_pol_map is a DMap object containing only the Q and U components of the input h_map
MPI.Scatter!(h_map, out_d_map, out_d_pol_map) #here out_d_map contains the I component, while out_d_pol_map Q and U 
````

Of course, the distribution of a polarized set of alms, represented in `Healpix.jl` by an `AbstractArray{Alm{T}, 1}`, works in a similar way: 
````julia
MPI.Scatter!(h_alms, out_d_pol_alms) #here both h_alms and out_d_pol_alms should only contain the E and B components
MPI.Scatter!(h_alms, out_d_alm, out_d_pol_alms) #here h_alms should contain [T,E,B], shared by out_d_alm (T) and out_d_pol_alm (E and B)
````

This allows the SHTs to be performed on the `DMap` and `DAlm` resulting objects directly, regardless of the field being polarized or not, as long as the number of components in the two objects is matching.
The functions `alm2map` and `adjoint_alm2map` will get authomatically the correct spin value for the given transform:
````julia
alm2map!(d_alm, d_map)         #spin-0 transform
alm2map!(d_pol_alm, d_pol_map) #polarized transform
````

## Run

In order to exploit MPI parallelization run the code through `mpirun` or `mpiexec` as
````shell
$ mpiexec -n {Ntask} julia {your_script.jl}
````

To run a code on multiple nodes, specify a machine file `machines.txt` as
````shell
$ mpiexec -machinefile machines.txt julia {your_script.jl}
````

## How to Cite

If you make use of HealpixMPI.jl for your work, please remember to cite it properly.

In order to do so, click on the *Cite this repository* menu in the *About* section of this repository.
Alternatively, use the following BibTeX entry:

```
@article{Bianchi_HealpixMPI_jl_an_MPI-parallel_2024,
author = {Bianchi, Leo A.},
doi = {10.21105/joss.06467},
journal = {Journal of Open Source Software},
publisher = {The Open Journal},
month = may,
number = {97},
pages = {6467},
title = {{HealpixMPI.jl: an MPI-parallel implementation of the Healpix tessellation scheme in Julia}},
url = {https://joss.theoj.org/papers/10.21105/joss.06467},
volume = {9},
year = {2024}
}
```
