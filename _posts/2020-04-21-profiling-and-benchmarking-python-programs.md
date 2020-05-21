---
layout: post
title: Profiling and benchmarking Python programs
date: 2020-04-21 17:09 +0900
---

The number of ways in which one can profile and benchmark Python programs
is daunting. There's many options out there, and this post is about the ones
that I found suitable for profiling and benchmarking PRs that I submit to
PyTorch every now and then. Coming from a land of C++ and Ruby, one annoying
thing I find about the Python tools is the preference for providing the
code to profiled inside a string as an argument to profiling tool, so
I try to directly instrument calls within the code wherever possible.

# Profiling C extensions

Say you want to know the function profiles of the following PyTorch script,
where we want to know where the `scatter\_` call is spending most of its time:
``` python
import torch
import numpy

M=256
N=512
dim = 0

input_one = torch.rand(M, N)
index = torch.tensor(numpy.random.randint(0, M, (M, N)))
res = torch.randn(M, N)

for _i in range(10000):
    res.scatter_(dim, index, input_one)
```

## Using cProfile

The default profiler for Python is `cProfile` which is a faster version of the `profile` module.
While this is simple to use and does not require any extra dependencies, it does not show profiles
of C++ functions at all. You can use it by calling the `cProfile.run` function and passing it
the code to be profiled as a string like so:
``` python
import cProfile

# Do something
cProfile.run("res.scatter_(dim,index,input_one)")
```
You can see the output of the profiler 

## Using yep

`yep`is a [utility](https://pypi.org/project/yep/) that uses Google's gperftools underneath and promises to
show profiles of C/C++ functions made inside Python C extensions. On Ubuntu/Debian, first install the `google-perftools`
package. Then run `pip install yep`.

You can set a region to profile as follows:
``` python
import yep

yep.start("file_name.prof")
# do something
yep.stop()
```
This generates a file `file_name.prof` that be can be analysed using the `pprof`
[utility](https://github.com/google/pprof) (which can be installed with `go get -u github.com/google/pprof`). You can
then get the top time consuming functions from `pprof` as follows:
```
pprof -text -lines file_name.prof
```
For our same program, profiling the `scatter_` loop shows the following output:
```
File: python3.6
Type: cpu
Showing nodes accounting for 27.51s, 98.81% of 27.84s total
Dropped 151 nodes (cum <= 0.14s)
      flat  flat%   sum%        cum   cum%
     4.45s 15.98% 15.98%     27.49s 98.74%  _ZZZZZN2at6native12_GLOBAL__N_130cpu_scatter_gather_base_kernelILb1EEclERNS_6TensorElRKS4_S7_RKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEEbRKNS0_17SCATTE
R_GATHER_OPEENKUlvE_clEvENKUlvE2_clEvENKUlRKT_E_clISt8functionIFvPfSR_EEEEDaSN_ENKUlPPcPKllE_clESV_SX_l /home/sameer/gitrepos/pytorch/build/aten/src/ATen/native/cpu/ScatterGatherKernel.cpp.AVX2.cpp:375
     2.84s 10.20% 26.19%      2.84s 10.20%  _ZNK2at6native12_GLOBAL__N_1UlPT_PT0_E2_clIffEEDaS3_S5_ /home/sameer/gitrepos/pytorch/build/aten/src/ATen/native/cpu/ScatterGatherKernel.cpp.AVX2.cpp:171
     2.54s  9.12% 35.31%      2.54s  9.12%  std::forward /usr/include/c++/7/bits/move.h:74
     1.91s  6.86% 42.17%      5.07s 18.21%  _ZNSt17_Function_handlerIFvPfS0_EN2at6native12_GLOBAL__N_1UlPT_PT0_E2_EE9_M_invokeERKSt9_Any_dataOS0_SE_ /usr/include/c++/7/bits/std_function.h:317
     1.39s  4.99% 47.16%     20.25s 72.74%  std::function::operator() /usr/include/c++/7/bits/std_function.h:706
     1.16s  4.17% 51.33%      1.16s  4.17%  std::forward /usr/include/c++/7/bits/move.h:73
     1.14s  4.09% 55.42%     11.48s 41.24%  _ZNSt17_Function_handlerIFvPfS0_EN2at6native12_GLOBAL__N_1UlPT_PT0_E2_EE9_M_invokeERKSt9_Any_dataOS0_SE_ /usr/include/c++/7/bits/std_function.h:316
     1.04s  3.74% 59.16%      1.04s  3.74%  _ZNSt14_Function_base13_Base_managerIN2at6native12_GLOBAL__N_1UlPT_PT0_E2_EE14_M_get_pointerERKSt9_Any_data /usr/include/c++/7/bits/std_function.h:176
     0.91s  3.27% 62.43%      0.91s  3.27%  _ZNSt14_Function_base13_Base_managerIN2at6native12_GLOBAL__N_1UlPT_PT0_E2_EE14_M_get_pointerERKSt9_Any_data /usr/include/c++/7/bits/std_function.h:175
     0.90s  3.23% 65.66%      0.90s  3.23%  std::_Any_data::_M_access /usr/include/c++/7/bits/std_function.h:107
     0.87s  3.12% 68.79%      0.87s  3.12%  _ZNK2at6native12_GLOBAL__N_1UlPT_PT0_E2_clIffEEDaS3_S5_ /home/sameer/gitrepos/pytorch/build/aten/src/ATen/native/cpu/ScatterGatherKernel.cpp.AVX2.cpp:170
     0.86s  3.09% 71.88%      0.86s  3.09%  std::function::operator() /usr/include/c++/7/bits/std_function.h:701
     0.79s  2.84% 74.71%      0.79s  2.84%  [libtorch_cpu.so]
```

## Some notes on yep

If you change the shared object file that your program was running and call `pprof` on the same `.prof` file,
the program will show nonsensical functions since it only maps the function hex code to the hex code from the 
shared object file.

# Analyzing performance regressions

Analysis of performance regressions requires comparing the same interfaces over different implementations.

## Time regression analysis

The simplest performance regression can be in terms of time of execution. Using the ipython magic command
is a great way to know mean and standard deviation of multiple executions of the same lines of code. Using
this within a script requires usage of embedded ipython. The `timeit` magic method allows for timing
code, and when used with the `-o` option will also return the object containing information about the
recent timing run.

# Further Reading

* C extentions with PySpy: https://www.benfrederickson.com/profiling-native-python-extensions-with-py-spy/
* Yep home page: https://pypi.org/project/yep/
* Speedscope homepage: https://github.com/jlfwong/speedscope
* Pyspy homepage: https://github.com/benfred/py-spy
* Google perftools: https://github.com/gperftools/gperftools
* Yep blog post:  https://www.camillescott.org/2013/12/06/yep/
* Timeit -o: https://ipython.readthedocs.io/en/stable/interactive/magics.html?highlight=timeit#magic-timeit
