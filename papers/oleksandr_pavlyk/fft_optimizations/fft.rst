Thin wrapper exposing Intel |R| MKL's FFT functionality directly on NumPy arrays was used to optimize FFT support in both NumPy and SciPy.
It allows Python programs relying on FFT computations approach performance of equivalent C implementations. 

Thanks to Intel |R| MKL's flexibility in its supports for arbitrarily strided input and output arrays [1]_ both one-dimensional and 
multi-dimensional complex Fast Fourier Transforms along distinct axes can be performed directly, without the need to copy the input 
into a contiguous array first. Input strides can be arbitrary, including negative or zero, as long strides remain an integer multiple 
of array's item size.

The wrapper supports both in-place and out-of-place modes, enabling it to efficiently power both `numpy.fft` and `scipy.fftpack` submodules. 
In-place operations are only performed where possible.

Direct support for multivariate transforms along distinct array axis. Even when multivariate transform ends up being computed as iterations 
of one-dimensional transforms, most iterations are performed in place for efficiency.

Dedicated support for complex FFTs on real inputs, such as `np.fft.fft(real_array)`.

Dedicated support for specialized real FFTs, which only store independent complex harmonics. Both `numpy.fft.rfft` and `scipy.fftpack.rfft` 
storage  modes are natively supported via Intel MKL.



References
----------


.. |C| unicode:: 0xA9 .. copyright sign
   :ltrim:
.. |R| unicode:: 0xAE .. registered sign
   :ltrim:
.. |TM| unicode:: 0x2122 .. trade mark sign
   :ltrim:

.. [1] https://software.intel.com/en-us/mkl-developer-reference-c-dfti-input-strides-dfti-output-strides#10859C1F-7C96-4034-8E66-B671CE789AD6
