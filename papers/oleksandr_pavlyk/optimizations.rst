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
NumPy is designed for high-performance basic arithmetic and transcendental operations on ndarrays. Some umath primitives are optimized to benefit from SSE, AVX and (recently) from AVX2 instruction sets, but not from AVX-512. Also, original NumPy functions did not take advantage of multiple cores. Update 2 provides substantial changes to the guts of NumPy to incorporate the Intel MKL Vector Math Library (VML) in respective umath primitives, which enables support for all available cores on a system and all CPU instruction sets.
The logic in Update 2 NumPy umath works as follows:
* For short NumPy arrays, the overheads to distribute work across multiple threads are high relative to the amount of computation work. In such cases, Update 2 uses the Intel MKL Short Vector Math Library (SVML), which is optimized for good performance across a range of Intel CPUs on short vectors.
* For large arrays, threading overheads are lower compared to the amount of computation and Update 2 uses the Intel MKL VML, which is optimized for utilizing multiple cores and a range of Intel CPUs.
NumPy Arithmetic and transcendental operations on vector-vector and vector-scalar are accelerated up to 400x for Intel |R| Xeon Phi |TM| processors.

Memory management optimizations
-------------------------------
Update 2 introduces widespread optimizations in NumPy memory management operations. As a dynamic language, Python manages memory for the user. Memory operations, such as allocation, de-allocation, copy, and move, affect performance of essentially all Python programs.
Specifically, Update 2 ensures NumPy allocates arrays that are properly aligned in memory on Linux, so that NumPy and SciPy compute functions can benefit from respective aligned versions of SIMD memory access instructions. This is especially relevant for Intel |R| Xeon Phi |TM| processors.
The most significant improvements in memory optimizations in Update 2 comes from replacing original memory copy and move operations with optimized implementations from Intel MKL. The result: improved performance because these Intel MKL routines are optimized for both a range of Intel CPUs and multiple CPU cores.

Faster Machine Learning with Scikit-learn
-----------------------------------------
Scikit-learn is well know library that provides a log of algorithms for many areas of machine learning.
Having limited developer resources this project prefers universal solutions and proven algorithms.
For performance improvement scikit-learn uses Cython and underling (through SciPy and Numpy) BLAS/LAPACK libraries.
OpenBLAS and MKL uses threaded based parallelism to utilize multicores of modern CPUs.
Unfortunately  BLAS/LAPACK’s functions are too low level primitives and their usage is often not very efficient comparing to possible high-level parallelism.
For high-level parallelism scikit-learn uses multiprocessing approach that is not very efficient from technical point of view.
On the other hand Intel provides Intel |R| Data Analytics Acceleration Library (Intel(R) DAAL) that helps speed up big data analysis by providing highly optimized algorithmic building blocks for all stages of data analytics (preprocessing, transformation, analysis, modeling, validation, and decision making) in batch, online, and distributed processing modes of computation.
It is originally written in C++ and provides Java and Python bindings.
In spite of the fact that DAAL is heavily optimized for all Intel Architectures including Xeon Phi it’s very questionable how to use DAAL binding from Python.
DAAL bindings for python are generated automatically and reflects original C++ API very close that makes its usage quite complicated because of not native for python programmers idioms and chary documentation.

To mix the power of well optimized for HW native code with familiar ML API Intel Python Distribuition started efforts on scikit-learn optimization.
Therefore, beginning from 2017 U2 IDP includes scikit-learn with daal4sklearn module.
Specifically, Update 2 optimizes Principal Component Analysis (PCA), Linear and Ridge Regressions, Correlation and Cosine Distances, and K-Means. Speedups may range from 1.5x to 160x.

There are no exact matching between sklearn and DAAL API and they aren’t fully compatible for all inputs so for the cases when daal4sklearn detects incompatibility it fallbacks to original sklearn’s implementation.

Daal4sklearn is enabled by default but provides simple API that allows disabling its functionality:

.. code-block:: python

        from sklearn.daal4sklearn import dispatcher
        dispatcher.disable()
        dispatcher.enable()

We prepared several benchmarks to demonstrate performance that can be achieved with DAAL.

.. code-block:: python

        from __future__ import print_function
        import numpy as np
        import timeit
        from numpy.random import rand
        from sklearn.cluster import KMeans

        import argparse
        argParser = argparse.ArgumentParser(prog="pairwise_distances.py",
                                            description="sklearn pairwise_distances benchmark",
                                            formatter_class=argparse.ArgumentDefaultsHelpFormatter)

        argParser.add_argument('-i', '--iteration', default='10', help="iteration", type=int)
        argParser.add_argument('-p', '--proc', default=-1, help="n_jobs for algorithm", type=int)
        args = argParser.parse_args()

        REP = args.iteration 

        try:
            from daal.services import Environment
            nThreadsInit = Environment.getInstance().getNumberOfThreads()
            Environment.getInstance().setNumberOfThreads(args.proc)
        except:
            pass

        def st_time(func):
            def st_func(*args, **keyArgs):
                times = []
                for n in range(REP):
                    t1 = timeit.default_timer()
                    r = func(*args, **keyArgs)
                    t2 = timeit.default_timer()
                    times.append(t2-t1)
                print (min(times), end='')
                return r
            return st_func

        problem_sizes = [
                (10000,   2),
                (10000,   25),
                (10000,   50),
                (50000,   2),
                (50000,   25),
                (50000,   50),
                (100000,  2),
                (100000,  25),
                (100000,  50)]

        X={}
        for p, n in problem_sizes:
            X[(p,n)] = rand(p,n)


        kmeans = KMeans(n_clusters=10, n_jobs=args.proc)
        @st_time
        def train(X):
            kmeans.fit(X)

        for p, n in problem_sizes:
            print (p,n, end=' ')
            X_local = X[(p,n)]
            train(X_local)
            print('')

Using all 32 cores of Xeon E5-2698v3 IDP’s KMeans can be faster more than 50 times comparing with python available on Ubuntu 14.04.
P below means amount of CPU cores used.

.. table:: 

 +------+----+---------+----------+------------+-------------+-------------+--------------+
 |rows  |cols|IDP,s P=1|IDP,s P=32|System,s P=1|System,s P=32|Vs System,P=1|Vs System,P=32|
 +======+====+=========+==========+============+=============+=============+==============+
 |10000 |2   |0,01     |0,01      |0,38        |0,27         |28,55        |36,52         |
 |10000 |25  |0,05     |0,01      |1,46        |0,57         |27,59        |48,22         |
 |10000 |50  |0,09     |0,02      |2,21        |0,87         |23,83        |40,76         |
 |50000 |2   |0,08     |0,01      |1,62        |0,57         |20,57        |47,43         |
 |50000 |25  |0,67     |0,07      |14,43       |2,79         |21,47        |38,69         |
 |50000 |50  |1,05     |0,10      |24,04       |4,00         |22,89        |38,52         |
 |100000|2   |0,15     |0,02      |3,33        |0,87         |22,30        |56,72         |
 |100000|25  |1,34     |0,11      |33,27       |5,53         |24,75        |49,07         |
 |100000|50  |2,21     |0,17      |63,30       |8,36         |28,65        |47,95         |
 +------+----+---------+----------+------------+-------------+-------------+--------------+

We compared the similar runs for other algorithms and normalized results by results obtained with DAAL in C++ without python to estimate overhead from python wrapping.


.. figure:: sklearn_perf.jpg 


You can find some benchmarks


.. _there: https://github.com/dvnagorny/sklearn_benchs


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
