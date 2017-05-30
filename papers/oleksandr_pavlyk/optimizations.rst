:author: Oleksandr Pavlyk
:email: Oleksandr.Pavlyk@intel.com
:institution: Intel Corporation
:corresponding:

:author: Denis Nagorny
:email: denis.nagorny@intel.com
:institution: Intel Corporation
:equal-contributor:

:author: Andres Guzman-Ballen
:email: andres.guzman-ballen@intel.com
:institution: Intel Corporation
:equal-contributor:

:author: Anton Malakhov
:email: Anton.Malakhov@intel.com
:institution: Intel Corporation
:equal-contributor:
:supervising-contributor:

:year: 2017

:video: http://www.youtube.com/watch?v=

-------------------------------------------------------
Accelerating Scientific Python with Intel Optimizations
-------------------------------------------------------

.. class:: abstract

    It is well known that the performance difference between python and basic C code can be 200x.
    Did you know that for numerically intensive code there is another 240x or more speedup possible?
    The performance comes from software’s ability to take advantage of your CPU's multiple cores,
    SIMD instructions, and high performance caches.
    This talk is for python programmers who want to get the most out of their hardware
    but do not have time or expertise to re-code their applications using native extensions,
    Cython, or Just-In-Time compilers that generate native code.

.. class:: keywords

   numpy,scipy,sk-learn,numba,simd,parall,optimization,performance

Introduction
------------

Intel released new version of its Distribution.. [TBC]
We suggest detailed report about optimization work we accomplished for Intel Distribution for Python,
which might be interesting for developers of SciPy tools.
It already offered great performance improvements for NumPy*, SciPy*, and Scikit-learn*
that you can see across a range of Intel processors,
from Intel |R| Core |TM| CPUs to Intel |R| Xeon |R| and Intel |R| Xeon Phi |TM| processors.
Here is the list of what we did for update 2:

Fast Fourier Transforms
-----------------------
In addition to initial Fast Fourier Transforms (FFT) optimizations offered in previous releases, Update 2 brings widespread optimizations for NumPy and SciPy FFT. It offers a layered interface for the Intel |R| Math Kernel Library (Intel |R| MKL) that allows efficient access to native FFT optimizations from a range of NumPy and SciPy functions. The optimizations include real and complex data types, both single and double precision. Update 2 covers both 1D and multidimensional data, in place and out of place. As a result, performance may improve up to 60x over Update 1 and is now close to native C/Intel MKL.

Arithmetic and transcendental expressions
-----------------------------------------
One of the great benefits found in our Intel® Distribution for Python is the performance boost gained from leveraging SIMD and multithreading in (select)
NumPy’s UMath arithmetic and transcendental operations, on a range of Intel CPUs, from Intel® Core™ to Intel® Xeon™ & Intel® Xeon Phi™. With stock python
as our baseline, we demonstrate the scalability of Intel® Distribution for Python by using functions that are intensively used in financial math applications
and machine learning:


One can see that stock Python (pip-installed NumPy from PyPI) on Intel® Core™ i5 performs basic operations such as addition, subtraction, and multiplication
just as well as Intel Python, but not on Intel® Xeon™ and Intel® Xeon Phi™, where Intel Python adds at least another 10x speedup. This can be explained by the
fact that basic arithmetic operations in stock NumPy are hard-coded AVX intrinsics (and thus already leverage SIMD, but do not scale to other ISA, e.g.
AVX-512). These operations in stock Python also do not leverage multiple cores (i.e. no multi-threading of loops under the hood of NumPy exist with such
operations). Intel Python’s implementation allows for this scalability by utilizing the following: respective Intel® MKL VML primitives, which are CPU-dispatched
(to leverage appropriate ISA) and multi-threaded (leverage multiple cores) under the hood, and Intel® SVML intrinsics, a compiler-provided short vector math
library that vectorizes math functions for both IA-32 and Intel® 64-bit architectures on supported operating systems. Depending on the problem size, NumPy will
choose one of the two approaches. On much smaller array sizes, Intel® SVML outperforms VML due to VML’s inherent cost of setting up the environment to multi-thread loops.
For any other problem size, VML outperforms SVML and this is thanks to VML’s ability to both vectorize math functions and multi-thread loops.


Specifically, on Intel® Core® i5 Intel Python delivers greater performance on transcendentals (log, exp, erf, etc.) due to utilizing both SIMD and multi-threading.
We do not see any visible benefit of multi-threading basic operations (as shown on the graph) unless NumPy arrays are very large (not shown on the graph). On Xeon®,
the 10x-1000x boost is explained by leveraging both (a) AVX2 instructions in transcendentals and (b) multiple cores (32 in our setup). Even greater scalability of
Xeon Phi® relative to Xeon is explained by larger number of cores (64 in our setup) and a wider SIMD.


The following charts provide another view of Intel Python performance versus stock Python on arithmetic and transcendental vector operations in NumPy by measuring
how close UMath performance is to respective native MKL call:
  
Again on Intel® Core™ i5 the stock Python performs well on basic operations (due to hard-coded AVX intrinsics and because multi-threading from Intel Python does not
add much on basic operations) but does not scale on transcendentals (loops with transcendentals are not vectorized in stock Python). Intel Python delivers performance
close to native speeds (90% of MKL) on relatively big problem sizes.



A Python implementation of the Black Scholes formula gives an idea of how NumPy UMath optimizations can be noticed on the application level:


One can see that on Intel® Core™ i5 the Black Scholes Formula scales nicely with Intel Python on small problem sizes but does not perform well on bigger problem sizes,
which is explained by small cache sizes. Stock Python does marginally scale due to leveraging AVX instructions on basic arithmetic operations, but this is a whole
different story on Intel® Xeon™ and Intel® Xeon Phi™. With Intel Python running the same Python code on server processors, much greater scalability on much greater
problem sizes is delivered. Intel® Xeon Phi™ scales better due to bigger number of cores and as expected, the stock Python does not scale on server processors due to
the lack of AVX2/AVX-512 support for transcendentals and no multi-threading utilization.


Memory management optimizations
-------------------------------
Update 2 introduces widespread optimizations in NumPy memory management operations. As a dynamic language, Python manages memory for the user. Memory operations, such as allocation, de-allocation, copy, and move, affect performance of essentially all Python programs.
Specifically, Update 2 ensures NumPy allocates arrays that are properly aligned in memory on Linux, so that NumPy and SciPy compute functions can benefit from respective aligned versions of SIMD memory access instructions. This is especially relevant for Intel |R| Xeon Phi |TM| processors.
The most significant improvements in memory optimizations in Update 2 comes from replacing original memory copy and move operations with optimized implementations from Intel MKL. The result: improved performance because these Intel MKL routines are optimized for both a range of Intel CPUs and multiple CPU cores.

Faster Machine Learning with Scikit-learn
-----------------------------------------
Scikit-learn is among the most popular Python machine learning packages. The initial release of Intel Distribution for Python provided Scikit-learn optimizations via respective NumPy and SciPy functions accelerated by Intel MKL. Update 2 optimizes selective key machine learning algorithms in Scikit-learn, accelerating them with the Intel |R| Data Analytics Acceleration Library (Intel |R| DAAL).
Specifically, Update 2 optimizes Principal Component Analysis (PCA), Linear and Ridge Regressions, Correlation and Cosine Distances, and K-Means. Speedups may range from 1.5x to 160x.

Numba vectorization
-------------------
We worked with Continuum Analytics to make Numba to vectorize math code with transcedential functions using Intel SVML library.


Numba Parallelism
-----------------
Intel Labs contributed Parallel Accelerator to Numba


Summary
-------
The Intel Distribution for Python is powered by Anaconda* and conda build infrastructures that give all Python users the benefit of interoperability within these two environments and access to the optimized packages through a simple conda install command.
Intel Distribution for Python 2017 Update 2 delivers significant performance optimizations for many core algorithms and Python packages, while maintaining the ease of download and install.


References
----------


.. |C| unicode:: 0xA9 .. copyright sign
   :ltrim:
.. |R| unicode:: 0xAE .. registered sign
   :ltrim:
.. |TM| unicode:: 0x2122 .. trade mark sign
   :ltrim:
