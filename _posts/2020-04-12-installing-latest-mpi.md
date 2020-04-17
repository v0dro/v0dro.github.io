---
layout: post
title: Installing latest MPI
date: 2020-04-12 16:23 +0900
---

Installing the latest openMPI can be a challenge if you want to correctly optimize
all parameters properly. Here are the right ways of doing so:

1. gdrcopy (https://github.com/NVIDIA/gdrcopy.git).Nothing special here, it figures out most of 
   he things. Do a `make INSTALL prefix=<somewhere GDR>`.
2. UCX (git@github.com:uccs/ucx.git). Pick the version you want (latest 1.8) and git checkout the
   corresponding branch. First `./autogen.sh` then
   `../../configure --prefix=<somewhere UCX> --disable-debug --with-cuda --with-avx --with-gdrcopy=<somewhere GDR> --enable-mt --with-hwloc`.
3. Finally OMPI (git@github.com:open-mpi/ompi.git). Similarly to UCX, pick the version you want
   (I stick with master most of the time except if it obviously broken), then `./autogen.sh` and
   then `../../configure --prefix=<somewhere OMPI> --enable-picky --disable-debug --enable-contrib-no-build=vt --enable-mpirun-prefix-by-default --with-cma --enable-ipv6 --disable-oshmem --disable-spc --with-ucx=<somewhere UCX> --with-cuda`

