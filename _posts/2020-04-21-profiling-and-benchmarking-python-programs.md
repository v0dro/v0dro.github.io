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

nfeat = 50
nrep = 2
res = torch.randn(1000, 1000)
batch = 200

c = torch.randint(3, (batch, nfeat * nrep)).float()
a = torch.arange(nfeat).repeat_interleave(nrep).unsqueeze(0).expand(batch,a.size(0))

res.scatter_(1,a,c)
```

## Using cProfile

The default

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
[utility](https://github.com/google/pprof) (which can be installed with `go get -u github.com/google/pprof`).

# Further Reading

* C extentions with PySpy: https://www.benfrederickson.com/profiling-native-python-extensions-with-py-spy/
* Yep home page: https://pypi.org/project/yep/
* Speedscope homepage: https://github.com/jlfwong/speedscope
* Pyspy homepage: https://github.com/benfred/py-spy
* Google perftools: https://github.com/gperftools/gperftools
