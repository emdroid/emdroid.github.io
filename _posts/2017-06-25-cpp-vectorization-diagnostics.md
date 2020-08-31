---
title: "C++ Vectorization Diagnostics"
header:
  teaser: /assets/images/thumb/speed.png
toc: true
categories:
  - Programming
  - C++
tags:
  - c++
  - optimization
  - vectorization
---

![Speed](/assets/images/thumb/speed.png){: .align-left .img-thumbnail}

**[Vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization)** is transformation of series of single operations into instructions which perform operations over multiple values simultaneously, by using the so-called *"[vector operations](https://en.wikipedia.org/wiki/Vector_processor) (or instructions)"* - MMX, SSE, AVX etc. (SIMD).

<!-- more -->

## 1. Introduction

Modern C++ compilers do these transformations automatically for speed-optimized builds, however the vectorization can be prevented by various reasons.
By default the compiler does not show any warnings/remarks in such case, and special compiler flags are needed to diagnose the vectorization status.

The automatic vectorization is usually performed upon loops, which process contiguous memory areas, such as vectors and arrays (matrices). For example, the loop:

```cpp
int A[1000], B[1000], C[1000];

// ... fill the A and B

// multiply the vectors
for (int i = 0; i < 1000; ++i)
    C[i] = A[i] * B[i];
```

is a trivial example ("[embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel)") which can be vectorized by using instructions performing four multiplications in parallel, equivalent to:

```cpp
int A[1000], B[1000], C[1000];

// ... fill the A and B

// multiply the vectors
for (int i = 0; i < 1000; i += 4) {
    // all 4 operation performed as a single instruction in parallel
    C[i+0] = A[i+0] * B[i+0];
    C[i+1] = A[i+1] * B[i+1];
    C[i+2] = A[i+2] * B[i+2];
    C[i+3] = A[i+3] * B[i+3];
}
```

The inability of the compiler to vectorize the loops can have various reasons - control flows in the loop (if branches, switches, breaks etc.), data dependencies (array overlaps, aliasing etc.). For example, this loop cannot be vectorized (filling the array with the Fibonacci numbers):

```cpp
int A[40];
A[0] = 0; A[1] = 1;
for (int i = 0; i < 38; ++i)
    A[i+2] = A[i] + A[i+1];
```

This is because in each iteration, the values used in the next steps are calculated.
So it cannot be transformed to perform multiple operations at contiguous areas at the same time, because the parallel operations would work upon wrong numbers:
If `A[3] = A[1] + A[2]` is performed simultaneously with `A[2] = A[0] + A[1]`, the `A[3]` would be calculated using a wrong (earlier) value of `A[2]`.

Note that the **compiler might vectorize the loops even in the case it cannot prove that the arrays do not actually overlap** (e.g. because of aliasing) - in such cases it might generate **both versions of the code** (vectorized and non-vectorized) and insert **runtime checks** to decide which version to use.
So the code might actually grow significantly, but it still should be (much) faster if the runtime checks succeed and the vectorized version can be used (which is the case most of the time, even though the compiler can't prove it at the compile time).

Also note that the automatic vectorization is in general **only performed when optimization is switched on** (`-O2`, `-O3` etc.).

See e.g. this video for some further details:

{% include video id="3MRxucTXPdw" provider="youtube" %}

## 2. Compiler support

### 2.1. Microsoft Visual Studio C++

The vectorization diagnostics in the Visual Studio compiler is printed out by using the **"[/Qvec-report](https://msdn.microsoft.com/en-us/library/jj614596.aspx)"** parameter (available since MSVC 2012).

Example code:

```cpp
#include <cstdlib>
#include <cstdio>

int main()
{
    int A[256], B[256], C[256];
    for (int i = 0; i != 256; ++i)
        A[i] = 1, B[i] = 2;

    for (int i = 0; i != 256; ++i)
        if (i < 128)
            C[i] = A[i] + B[i];
        else
            C[i] = B[i] * A[i];

    printf("%d,%d,%d,%d\n", C[0], C[127], C[128], C[255]);
    return EXIT_SUCCESS;
}
```

When trying to compile this source code:

```
cl /nologo /EHsc /O2 /Qvec-report:2 vectorize.cpp

--- Analyzing function: main
vectorize.cpp(11) : info C5002: loop not vectorized due to reason '1100'
```

This shows that the loop could not be vectorized, along with the reason code.
All the reason codes are listed here:
[Vectorizer and Parallelizer Messages](https://msdn.microsoft.com/en-us/library/jj658585.aspx). 
In this particular case, the reason is the `"if"` control statement inside the loop.

This can be fixed by splitting the loop into two separate loops, avoiding the `"if"` statement:

```cpp
#include <cstdlib>
#include <cstdio>

int main()
{
    int A[256], B[256], C[256];
    for (int i = 0; i != 256; ++i)
        A[i] = 1, B[i] = 2;


    for (int i = 0; i != 128; ++i)
        C[i] = A[i] + B[i];

    for (int i = 128; i != 256; ++i)
        C[i] = B[i] * A[i];

    printf("%d,%d,%d,%d\n", C[0], C[127], C[128], C[255]);
    return EXIT_SUCCESS;
}
```

After this change:

```
cl /nologo /EHsc /O2 /Qvec-report:2 vectorize.cpp

--- Analyzing function: main
vectorize.cpp(11) : info C5001: loop vectorized
vectorize.cpp(14) : info C5001: loop vectorized
```

Be prepared however, that if the STL headers are used (e.g. "*iostream*", "*algorithm*"), they might generate a lot of vectorization diagnostic messages because of the loops used in there, which often are not vectorized (this is in particular often an issue when iterators are used).

### 2.2. GNU C++ compiler

The GCC compiler flags to print out the vectorization diagnostics are "**-ftree-vectorizer-verbose=n**" ("n" = the verbosity level) and/or "**-fopt-info-vec-missed**".
To actually switch the automatic vectorization on, either the `"-O3"` flag or `"-O2"` with `"-ftree-vectorize"` need to be used.

In some cases the architecture or support of the particular vector instructions need to be selected, such as: `"-march=native"`, `"-match=corei7"`, `"-msse2"`, `"-mavx"` etc.
Be aware that such executable will then only run on the machines which support the selected architecture options.

With the same example as for the MSVC compiler:

```cpp
#include <cstdlib>
#include <cstdio>

int main()
{
    int A[256], B[256], C[256];
    for (int i = 0; i != 256; ++i)
        A[i] = 1, B[i] = 2;

    for (int i = 0; i != 256; ++i)
        if (i < 128)
            C[i] = A[i] + B[i];
        else
            C[i] = B[i] * A[i];

    printf("%d,%d,%d,%d\n", C[0], C[127], C[128], C[255]);
    return EXIT_SUCCESS;
}
```

```
g++ -O3 -ftree-vectorizer-verbose=1 -o vectorize vectorize.cpp

Analyzing loop at vectorize.cpp:11

Analyzing loop at vectorize.cpp:7

Vectorizing loop at vectorize.cpp:7

vectorize.cpp:7: note: LOOP VECTORIZED.
vectorize.cpp:4: note: vectorized 1 loops in function.
```

The loop we care about (the second one) was not vectorized.
To show further diagnostics the level can be increased:

```
g++ -O3 -ftree-vectorizer-verbose=2 -o vectorize vectorize.cpp

Analyzing loop at vectorize.cpp:11

vectorize.cpp:11: note: not vectorized: control flow in loop.
vectorize.cpp:11: note: bad loop form.
Analyzing loop at vectorize.cpp:7

vectorize.cpp:7: note: misalign = 0 bytes of ref A[i_33]
vectorize.cpp:7: note: misalign = 0 bytes of ref B[i_33]
vectorize.cpp:7: note: virtual phi. skip.

Vectorizing loop at vectorize.cpp:7

vectorize.cpp:4: note: vectorized 1 loops in function.
...
```

(the level can be increased even more to see further diagnostics)

So the reason the loop has not been vectorized is the control flow (`"if"`) in the loop.

Now when the loop is split into two separate loops:

```cpp
#include <cstdlib>
#include <cstdio>

int main()
{
    int A[256], B[256], C[256];
    for (int i = 0; i != 256; ++i)
        A[i] = 1, B[i] = 2;

    for (int i = 0; i != 128; ++i)
        C[i] = A[i] + B[i];

    for (int i = 128; i != 256; ++i)
        C[i] = B[i] * A[i];

    printf("%d,%d,%d,%d\n", C[0], C[127], C[128], C[255]);
    return EXIT_SUCCESS;
}
```

```
g++ -O3 -ftree-vectorizer-verbose=1 -o vectorize vectorize.cpp

Analyzing loop at vectorize.cpp:14

Vectorizing loop at vectorize.cpp:14

vectorize.cpp:14: note: LOOP VECTORIZED.
Analyzing loop at vectorize.cpp:11

Vectorizing loop at vectorize.cpp:11

vectorize.cpp:11: note: LOOP VECTORIZED.
Analyzing loop at vectorize.cpp:7

Vectorizing loop at vectorize.cpp:7

vectorize.cpp:7: note: LOOP VECTORIZED.
vectorize.cpp:4: note: vectorized 3 loops in function.
```

Now all the loops were successfully vectorized.

### 2.3 CLang (LLVM) compiler

The CLang compiler has the following options for vectorization diagnostics (only supported in newer CLang versions):

- **-Rpass=loop-vectorize**: Identifies successfully vectorized loops.
- **-Rpass-missed=loop-vectorize**: Identifies loops that failed to vectorize.
- **-Rpass-analysis=loop-vectorize**: Shows the statements that caused the vectorization to fail.

To switch the vectorization on, again the `"-O3"` or "`-O2 -ftree-vectorize``" flags need to be used.

Normally the `"-Rpass=loop-vectorize"` is not necessary, so that in the case everything was successfully vectorized no diagnostics is printed.

With the first example:

```cpp
#include <cstdlib>
#include <cstdio>

int main()
{
    int A[256], B[256], C[256];
    for (int i = 0; i != 256; ++i)
        A[i] = 1, B[i] = 2;

    for (int i = 0; i != 256; ++i)
        if (i < 128)
            C[i] = A[i] + B[i];
        else
            C[i] = B[i] * A[i];

    printf("%d,%d,%d,%d\n", C[0], C[127], C[128], C[255]);
    return EXIT_SUCCESS;
}
```

```
clang++ -O3 -Rpass=loop-vectorize -Rpass-missed=loop-vectorize -Rpass-analysis=loop-vectorize -o vectorize vectorize.cpp

vectorize.cpp:7:5: remark: vectorized loop (vectorization width: 4, interleaved count: 2) [-Rpass=loop-vectorize]
    for (int i = 0; i != 256; ++i)
    ^
vectorize.cpp:11:15: remark: the cost-model indicates that interleaving is not beneficial
      [-Rpass-analysis=loop-vectorize]
        if (i < 128)
              ^
vectorize.cpp:11:15: remark: vectorized loop (vectorization width: 4, interleaved count: 1) [-Rpass=loop-vectorize]
```

Note that the CLang optimizer was still able to vectorize the second loop, however prints a remark on the `"if"` statement line, and the "*interleaved count*" is only 1 instead of 2. The interleaved instructions can exploit advanced hardware features, such as multiple execution units and out-of-order execution.

The loop split into two separate loops:

```cpp
#include <cstdlib>
#include <cstdio>

int main()
{
    int A[256], B[256], C[256];
    for (int i = 0; i != 256; ++i)
        A[i] = 1, B[i] = 2;

    for (int i = 0; i != 128; ++i)
        C[i] = A[i] + B[i];

    for (int i = 128; i != 256; ++i)
        C[i] = B[i] * A[i];

    printf("%d,%d,%d,%d\n", C[0], C[127], C[128], C[255]);
    return EXIT_SUCCESS;
}
```

```
clang++ -O3 -Rpass=loop-vectorize -Rpass-missed=loop-vectorize -Rpass-analysis=loop-vectorize -o vectorize vectorize.cpp
vectorize.cpp:7:5: remark: vectorized loop (vectorization width: 4, interleaved count: 2) [-Rpass=loop-vectorize]
    for (int i = 0; i != 256; ++i)
    ^
vectorize.cpp:11:16: remark: vectorized loop (vectorization width: 4, interleaved count: 2) [-Rpass=loop-vectorize]
        C[i] = A[i] + B[i];
               ^
vectorize.cpp:14:16: remark: vectorized loop (vectorization width: 4, interleaved count: 2) [-Rpass=loop-vectorize]
        C[i] = B[i] * A[i];
               ^
```

All the loops were optimally vectorized now.

See here for further information about the CLang/LLVM: [Loop Vectorization: Diagnostics and Control](http://blog.llvm.org/2014/11/loop-vectorization-diagnostics-and.html)

{% include abbrev domain="computers" %}
