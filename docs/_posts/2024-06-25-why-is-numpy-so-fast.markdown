---
layout: post
title:  Why is Numpy so fast?
date:   2024-06-25
categories: "deep dives"
pagination: enabled
---


# Why is Numpy so fast?
### Jun 25, 2024

I recently came across a video that was using clever Numpy to accelerate some fractal rendering code.
On first thought it's very easy to guess why it's fast - C! But there's more reasons: 

## Homogeneity
Numpy arrays have elements with homogeneous types, whilst native Python lists are just containers holding pointers to objects - even when they are of the same type. 
The [Principle of Locality](https://en.wikipedia.org/wiki/Locality_of_reference) is the tendency of a processor to access the same set of memory locations repetitively over a short period of time. Thus, since Numpy arrays are homogeneous, these homogeneous elements can be cached, and future accesses to these will be relatively faster.
Numpy arrays are contiguous. This means that the processor just loads the entire block into cache. So, accesses to all elements within the block are faster.

Due to this homogeneity, a lot of latency is saved on [pointer indirection](https://en.wikipedia.org/wiki/Indirection) and per-element type checking.

## Vectorized operations
Arithmetic operations are applied to the entire array at once, instead of having to explicitly loop over the latter just to access an element, which introduces a complexity of $O(N)$ at worst. Numpy just offloads array processing to C, so array operations like iterations should always be done in such vectorized operations.

Array broadcasting is done when arrays are of different sizes, so it becomes possible to perform arithmetic operations between arrays are scalars. The scalar value is essentially expanded - so the number 2 will be treated as an array filled with twos. This is done so that looping occurs in C instead of Python. Needless copies of data are not created, so efficiency is almost always a by-product. 

Broadcasting (like many other vectorized operations), under the hood is almost always done in the form of "Universal functions" (ufunc for short). These are functions that operate on `ndarrays` in an element by element fashion. Most of the built-in functions, like addition are carried out this way.

## Efficient internal organization
Numpy arrays aren't *actually* arrays. They're a data structure that consists of a contiguous data buffer and some metadata. Let's talk a bit more in detail about the internal implementation.

#### Internal data buffer
It is essentially a C-style array - a contiguous, fixed block of memory, containing similar elements. This is the portion that actually holds the array elements. 

This is the fundamental aspect of an `ndarray` - It is essentially a chunk of memory starting at some location. 

#### Buffer Metadata
It contains the following (quoted directly from the Numpy documentation):
- Size of the basic data element (ints are 4 bytes, irrespective of 32-bit or 64-bit systems)
- an offset relative to the start of data buffer
- the number of dimensions and the size of each dimension
- the [stride](https://en.wikipedia.org/wiki/Stride_of_an_array)
- [byte order](https://en.wikipedia.org/wiki/Endianness)
- read flag (whether the buffer is read-only, or not)
- dtype: interpretation of the basic data element (yes, the users can create [arbitrarily complex, composite data types](https://numpy.org/doc/stable/glossary.html#term-structured-data-type) as the basic array element!)
- order of the array: C, or Fortran

If you want to read more about how Numpy works under the good, check out [Guide to Numpy](https://www.amazon.com/Guide-NumPy-Travis-Oliphant-PhD/dp/151730007X) 
by Travis Oliphant, who's credited with the proliferation of Numpy, Scipy, and even Python itself.

# Some Benchmarking
Let's see how fast Numpy is. I'll use the dot product of two NxN matrices as reference.
The elements of both the matrices will be randomly generated using `numpy.random.rand()`. Intuitively,
the random element generation was excluded from the time checks.

The benchmarks were done on an IPython Kernel, with an Intel Xeon(R). The `timeit` console utility
was used to time function calls. The Python documentation for [timeit](https://docs.python.org/3/library/timeit.html) 
states that there is a certain base overhead for executing a pass statement. 
You can check that overhead for your machine by invoking `timeit` without any arguments. 

<!-- Numpy benchmark photo -->


I also want to compare Numpy to BLAS and LAPACK, as Numpy does rely on both of them 
for some operations if they are installed. Benchmarking is a tough subject though, so I'll do extensive benchmarking in the next
    article in this series. There, I'll also compare it to other Linear Algebra APIs, maybe in C/C++.

<!-- Read more about BLAS and LAPACK with Numpy [here](https://superfastpython.com/what-is-blas-and-lapack-in-numpy/). -->

All of the code for benchmarking is published [here](). It's not perfect, but it certainly gives an idea.


# What next?
It is important to note that Numpy is not *always* fast. I'll talk more about Numpy specific use cases, 
and where it fails against Vanilla Python, and compare Numpy's performance to BLAS and LAPACK and C++ 
in the next article in this series.
