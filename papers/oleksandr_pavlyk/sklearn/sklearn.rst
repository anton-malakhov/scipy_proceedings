Scikit-learn is well know library that provides a lot of algorithms for many areas of machine learning.
Having limited developer resources this project prefers universal solutions and proven algorithms.
For performance improvement scikit-learn uses Cython and underlying BLAS/LAPACK libraries through SciPy and Numpy.
OpenBLAS and MKL uses threaded based parallelism to utilize multicores of modern CPUs.
Unfortunately  BLAS/LAPACK’s functions are too low level primitives and their usage is often not very efficient comparing to possible high-level parallelism.
For high-level parallelism scikit-learn uses multiprocessing approach that is not very efficient from technical point of view.
On the other hand Intel provides Intel |R| Data Analytics Acceleration Library (Intel |R| DAAL) that helps speed up big data analysis by providing highly optimized algorithmic building blocks for all stages of data analytics (preprocessing, transformation, analysis, modeling, validation, and decision making) in batch, online, and distributed processing modes of computation.
It is originally written in C++ and provides Java and Python bindings.
DAAL is heavily optimized for all Intel Architectures including Xeon Phi, but it is not at all clear how to use DAAL binding from Python.
DAAL bindings for python are generated automatically and reflects original C++ API very closely. This makes its usage quite complicated because of its use of non pythonic idioms and scarce documentation.

In order to combine the power of well optimized native code with the familiar to machine learning community API the Intel Distribution for Python includes fruits of efforts of scikit-learn optimization. Thus beginning with Intel Distribution for Python 2017.0.2 includes scikit-learn with daal4sklearn sub-module.
Specifically, update 2 optimizes Principal Component Analysis (PCA), Linear and Ridge Regressions, Correlation and Cosine Distances, and K-Means. Speedups may range from 1.5x to 160x.

There are no direct matching between sklearn's and DAAL's APIs and they aren’t fully compatible for all inputs, therefore in cases when daal4sklearn detects incompatibility it fallbacks to original sklearn’s implementation.

Daal4sklearn is enabled by default but provides simple API that allows disabling its functionality:

.. code-block:: python

        from sklearn.daal4sklearn import dispatcher
        dispatcher.disable()
        dispatcher.enable()

We prepared several benchmarks to demonstrate performance that can be achieved with DAAL.

.. code-block:: python

        from __future__ import print_function
        import timeit
        from numpy.random import rand
        from sklearn.cluster import KMeans

        import argparse
        argParser = argparse.ArgumentParser(
            prog="kmeans_bench.py",
            description="K-means benchmark")

        argParser.add_argument('-i', '--iteration',
              help="iteration", type=int, default=10)
        argParser.add_argument('-p', '--proc', default=-1,
              help="n_jobs for algorithm", type=int)
        args = argParser.parse_args()

        try:
            from daal.services import Environment
            env_inst = Environment.getInstance()
            nThreadsInit = env_inst.getNumberOfThreads()
            env_inst.setNumberOfThreads(args.proc)
        except:
            pass

        def st_time(func):
            def st_func(*args, **keyArgs):
                times = []
                for n in range(args.iteration):
                    t1 = timeit.default_timer()
                    r = func(*args, **keyArgs)
                    t2 = timeit.default_timer()
                    times.append(t2-t1)
                print (min(times), end='')
                return r
            return st_func

        problem_sizes = [
                (10000,  2),  (10000,  25),
                (10000,  50), (50000,  2),
                (50000,  25), (50000,  50),
                (100000, 2),  (100000, 25),
                (100000, 50)]
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

Using all 32 cores of Xeon E5-2698 v3 IDP’s K-Means can be faster more than 50 times comparing with python available on Ubuntu 14.04.
P below means the number of CPU cores used.

.. table:: 
   :class: w

   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | rows   | cols | IDP,s P=1 | IDP,s P=32 | System,s P=1 | System,s P=32 | Vs System,P=1 | Vs System,P=32 | 
   +========+======+===========+============+==============+===============+===============+================+
   | 10000  | 2    | 0.01      | 0.01       | 0.38         | 0.27          | 28.55         | 36.52          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | 10000  | 25   | 0.05      | 0.01       | 1.46         | 0.57          | 27.59         | 48.22          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | 10000  | 50   | 0.09      | 0.02       | 2.21         | 0.87          | 23.83         | 40.76          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | 50000  | 2    | 0.08      | 0.01       | 1.62         | 0.57          | 20.57         | 47.43          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | 50000  | 25   | 0.67      | 0.07       | 14.43        | 2.79          | 21.47         | 38.69          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | 50000  | 50   | 1.05      | 0.10       | 24.04        | 4.00          | 22.89         | 38.52          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | 100000 | 2    | 0.15      | 0.02       | 3.33         | 0.87          | 22.30         | 56.72          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | 100000 | 25   | 1.34      | 0.11       | 33.27        | 5.53          | 24.75         | 49.07          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+
   | 100000 | 50   | 2.21      | 0.17       | 63.30        | 8.36          | 28.65         | 47.95          | 
   +--------+------+-----------+------------+--------------+---------------+---------------+----------------+

We compared the similar runs for other algorithms and normalized results by results obtained with DAAL in C++ without python to estimate overhead from python wrapping.


.. figure:: sklearn/sklearn_perf.jpg


You can find some benchmarks [sklearn_benches]_


.. [sklearn_benches] https://github.com/dvnagorny/sklearn_benchs

