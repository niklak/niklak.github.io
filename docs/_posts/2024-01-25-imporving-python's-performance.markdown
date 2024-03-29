---
layout: post
title:  "Improving python's performance"
date:   2024-01-25 10:23:43 +0300
categories: python
---

Let's consider a situation, almost straight from real life – you had a `Python` codebase performing some heavy-duty tasks. Over time, either your management or you began to suspect that this code might not be as efficient as expected – it either doesn't run as fast or consumes an excessive amount of RAM.

At this point, it's appropriate to make a pointed remark like *"You should write better code!"* And indeed, we should always strive to write better code when justified by time and resources.

It's essential to remember that `Python` is a fast language in terms of development but not always in terms of performance. Until recently, `python`'s performance fell behind languages like `JavaScript`, `Ruby`, `PHP`, and others that operate through interpreters. The situation has improved, and it will likely continue to do so with each release. However, it doesn't mean that other programming languages will stand still. So, it's an ongoing race.

While `Python` can be used to write performant code, writing such code is more challenging. The main goal of those developing `Python` is to simplify the writing of efficient code.
So, let's work with what we have – `Python` and its capabilities. It's unnecessary to optimize everything. The key is to identify the bottleneck affecting performance. There are several tools for this purpose. You can start with `memory_profiler`. If you're not fond of `memory_profiler`, there are alternatives – plenty of them. I hope you are not lazy and able to use your favorite search engine.

So let's consider the following options. 

### The problem is in our code

#### Replacing Complex Structures with Simpler Ones, If Possible.

For example, if our functions are creating too many dictionaries/sets within. If so, it might be useful to create benchmarks to compare the costs of this function with alternatives, without dictionaries/sets. If a dictionary, set, or another complex structure is created many times and is not significantly large, it makes sense to use a simpler and more cost-effective structure instead. 

#### Caching frequent functions.

Suppose a specific function takes a list of non-unique numbers with many duplicates as an argument. Then a complex calculation is applied to each element of the list. In this case, you may use `functools.lru_cache`. Of course, it is just one of the many ways to solve such a problem.

### Replacing project packages with optimized versions of it.

Many people were concerned about the performance of `Python` before us, and much has already been optimized. For example, if we observe (thanks to profiling) that our service spends a lot of time on encoding/decoding `JSON`, we should look for an alternative high-performance package. For instance, `orjson` or `msgspec`. Typically, this code is written in a performant programming language and includes a wrapper for `Python`.


### Code Transformation into Multiprocessing

If the primary goal of the code is CPU-bound operations, then you can consider the multiprocessing module. It makes it possible to share the execution of tasks among processes.

### Using NumPy and Numba

If your service/program involves complex matrix calculations, it's worth considering the use of `NumPy` and `Numba`. If you have a videocard with CUDA support then you can replace `NumPy` with `CuPy`.

### Using Cython
As an alternative to the previous option you can rewrite part of the code in `Cython`.

### Running the Script on PyPy

This is my favorite option. `PyPy` has its nuances, but generally, the entire code will work faster immediately without any changes or with minimal modifications.

### Using asyncio for I/O Tasks
If it's a reasonably serious application, it's hard to imagine not using `asyncio` for I/O tasks (web applications, database connections, service connections, etc.). The capabilities of the `asyncio` package can significantly improve the performance of your project.


### Delegating Tasks with Slow Code to External Execution

#### Using gRPC

This is a good option, and it's not complex, although you might have to tinker with it. The benefit is that a significant portion of the code for `gRPC` will be generated by special utilities. The hardest part is to rewrite the problematic code with a performant language, for example, `Go` or `Rust` (or `C`/`C++`, if you prefer). Then, you'll need to deploy your `gRPC` server and connect it to your `Python` code.

#### Using FFI

The essence is similar to `gRPC`, i.e., calling functions from an external performant environment. However, I like it less because there may be potential issues with it. It's crucial to thoroughly check for various memory leaks. Still, it can be a good option if you have at least some knowledge of `Rust` or wish to explore it. This is because `Rust` has several excellent packages for calling `Rust` code from Python – `PyO3`, `maturin`, and `uniffi`.

In any case, both options are good because without changing the logic but only the execution environment, you already gain significant performance improvement.

### Radical option: rewrite all your Python code with Go.

This is not a joke. It's also a good solution. `Go` is simpler than Python, and it's not just my opinion. I've seen people who were confused by OOP, `asyncio`, and `multiprocessing` in `Python`. In `Go`, these things didn't distract them. `Go` is simpler and faster than Python but comes with some drawbacks that need to be taken into account. Packages in `Go` are not as reliable as in Python. Even the standard package `net/url` might surprise you at some point.  Additionally, `go` lacks a good dependency manager, making comparisons with managers in other languages somewhat pointless.

### Conclusion

You can enhance the performance of `Python` using the mentioned options as well as others, but everything has its limits. `Python`'s inherent limit may seem modest, but the existing possibilities allow expanding this limit by incorporating features from other programming languages.
