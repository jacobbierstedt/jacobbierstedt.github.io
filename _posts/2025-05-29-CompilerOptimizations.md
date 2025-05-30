---
layout: post
title:  "Understanding compiler optimizations with assembly"
date:   2025-05-29
categories: blog
---

Some bioinformatics engineers spend a lot of time and energy making their tools more efficient. When this time is spent designing the overall algorithms and data structures, this can be extremely advantageous. However, when trying to make low-level optimizations on single operations, the compiler is usually going to optimize these and the programmer should instead focus on making the code readable and easier to maintain.

As a bioinformatician without formal training in computer science, there is a lot that I don't know about compilers and optimization, but here is a nonexhaustive list with some key optimizations with the assembly that I find particularly helpful to understand in creating efficient software. The beauty of it is that, apart from the design considerations, many of these are done by the compiler by simply enabling optimizations.

## Inspecting compiler output
The best way to understand what is happening in your code is to look at the assembly. There are good, readily-available tools for understanding compiler output:
1. Compile with `gcc -fopt-info`
2. [godbolt.org](https://godbolt.org)
3. `objdump -d -M intel <binary>`

## General considerations
It is far more important to design software that is algorithmically efficient. It is extremely common in bioinformatics that this gets overlooked and users end up throwing compute at the problem. Do you really need 40 CPUs and 500Gb RAM? Sometimes, but not usually. If this is the case, you need to look at your design and memory usage, the below optimizations will not help. These are just for eking out the last bit of performance and, more than anything, they are helpful for understanding what is actually going on in your code.

## Inlining
Function inlining is a simple optimization that replaces the function call with the body of the function itself. This can certainly save some time, but at the expense of a larger binary. In some cases, the compiler eliminates the function calls and the stack operations that are associated with them. For example:
```c
int square(int x) {
  return x * x;
}

int add_and_square(int x, int y) {
  return square(x) + y;
}
```
If we compile these without inlining, we get the following assembly:
```nasm
square:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     eax, DWORD PTR [rbp-4]
        imul    eax, eax
        pop     rbp
        ret
add_and_square:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 8
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     eax, DWORD PTR [rbp-4]
        mov     edi, eax
        call    square
        mov     edx, DWORD PTR [rbp-8]
        add     eax, edx
        leave
        ret
```
In both functions, the first instruction is `push    rbp` which saves the base pointer of the caller function, followed by the creation of a new frame on the stack for the callee: `mov   rbp, rsp`. After the body of the function, we restore the base pointer of the caller and return. At least 4 instructions just for calling the function. 

In the function `add_and_square` we are calling the function `square`. If we let the compiler optimize, with `-O1` or above, we will see something like this:
```nasm
square:
        imul    edi, edi
        mov     eax, edi
        ret
add_and_square:
        imul    edi, edi
        lea     eax, [rdi+rsi]
        ret
```
First, there is no stack setup or teardown for our functions in either case, but that is out of scope for this example. The important detail is that the compiler has automatically inlined `square` and replaced the call to `square` with the instruction `imul    edi, edi`. This takes the first argument and multiplies it by itself.

While they aren't necessarily huge, one can easily imagine the benefits of this when small functions are called many times. Of course, a programmer could do some amount of inlining themselves but at the expense of readability and maintainability.

The `inline` keyword is a suggestion and doesn't guarantee that your function will be inlined. Besides looking at the assembly dump, one way to check is to compile with `-fopt-info` and inspect the output.


## Automatic vectorization of loops
Often times, the compiler will attempt to recognize loops that can be vectorized and compile them to SIMD instructions using vector registers. This can give a huge performance boost.

An example of a loop that can be auto vectorized:
```c
void auto_vec(int* c, int* b, int* a, int n) {
  for (int i = 0; i < n; i++) {
    c[i] = b[i] + a[i];
  }
}
```
If we compile this with `-O2`, we get a simple loop:
```nasm
auto_vec:
        mov     r8, rdx
        test    ecx, ecx
        jle     .L1
        movsx   rcx, ecx
        xor     eax, eax
        sal     rcx, 2
.L3:
        mov     edx, DWORD PTR [r8+rax]
        add     edx, DWORD PTR [rsi+rax]
        mov     DWORD PTR [rdi+rax], edx
        add     rax, 4
        cmp     rcx, rax
        jne     .L3
.L1:
        ret
```
Initial checks are performed and then it moves into to the main body of the loop, `.L3`. We move `a[i]` into `edx`, add `b[i]` to `edx`, and move `edx` to `c[i]`. We increment i (`add    rax, 4`) and check if the loop is done.


If we compile with `gcc -O2 -ftree-vectorize` we can look at the godbolt output (full output at the bottom of the page), we can see that there is a lot more going on. The scalar loop from above is there as a runtime fallback if for any reason the loop can't be vectorized (`.L3` `.L9`), and there is additional SIMD instructions with vector registers to optimize where we can.

```nasm
.L5:
        movdqu  xmm0, XMMWORD PTR [rsi+rax]
        movdqu  xmm2, XMMWORD PTR [rdx+rax]
        paddd   xmm0, xmm2
        movups  XMMWORD PTR [rdi+rax], xmm0
        add     rax, 16
        cmp     r8, rax
        jne     .L5
```
The `xmm0` and `xmm2` are 128-bit registers that are used in this case to hold 4 integers from `a` and `b` at a time. We add them with `paddd` and then store the result in `c`. In this example, each iteration of the loop processes 4 elements from `a` and `b` simultaneously.

This can provide a significant performance boost. To check if your compiler is vectorizing your loops:
1. Ensure that you are using the right flags. If gcc, at least `-O2` or `-ftree-vectorize`
2. Inspect the output when compiling with `-fopt-info-vec -fopt-info-vec-missed` to get detailed summary of what was optimized, missed, and why.

## Loop unrolling
Loop unrolling can provide a performance boost by processing the data in chunks and using fewer loop related instructions. For example, we can (but shouldn't) manually unroll this loop by using a larger step size with something like this:

```c
void add_ints(int* x, int* y, int* z, int n) {
  for (int i = 0; i < n; i++) {
    z[i] = x[i] + y[i];
  }
}

void unrolled_add_ints(int* x, int* y, int* z, int n) {
    int i;
    for (i = 0; i <= n - 4; i += 4) {
        z[i]     = x[i]     + y[i];
        z[i + 1] = x[i + 1] + y[i + 1];
        z[i + 2] = x[i + 2] + y[i + 2];
        z[i + 3] = x[i + 3] + y[i + 3];
    }
}
```
If we inspect the assembly for each of these with `-O3 -funroll-loops` we will see that the compiler optimizes the first with SIMD vectorization and unrolled loops. 4 elements of the arrays are added at a time, and the loop progresses in step sizes of 4. This can have a significant impact on execution time when dealing with a large amount of data. It also is a strong piece of evidence that the programmer should trust the compiler, as the manually unrolled loop is not pretty to read.

## Integer swap without a temp variable
Sometimes what you think you are doing and what the compiler actually does is different. A classic example that illustrates this (and common interview question) is the difference between using a third variable vs XOR to swap two ints. The first example uses a temp variable to swap the two and the second example uses the xor. 
```c
void swap(int* a, int* b) {
  int c = *a;
  *a = *b;
  *b = c;
}

void xor_swap(int* a, int* b) {
  *a ^= *b;
  *b ^= *a;
  *a ^= *b;
}
```
To compare these two implementations of the same thing, we can look at the assembly generated with `-O1` or higher.
```nasm
swap:
        mov     eax, DWORD PTR [rdi]
        mov     edx, DWORD PTR [rsi]
        mov     DWORD PTR [rdi], edx
        mov     DWORD PTR [rsi], eax
        ret
xor_swap:
        mov     eax, DWORD PTR [rdi]
        xor     eax, DWORD PTR [rsi]
        mov     DWORD PTR [rdi], eax
        xor     eax, DWORD PTR [rsi]
        mov     DWORD PTR [rsi], eax
        xor     DWORD PTR [rdi], eax
        ret
```
The compiler has optimized `swap` to 4 instructions and `xor_swap` to 6. In `swap`, the compiler is using two registers (`eax` and `edx`) to load `a` and `b`. It then stores the values from `eax` and `edx` to the memory pointed to by `a` and `b`. In the `xor_swap`, we are using a single register and doing 6 operations involving memory reading and writing. While it might be a neat trick, XOR swap is slower, fragile, much harder to read, and uses a register as a "third temp variable" anyways.



### Complete `auto_vec` assembly
```nasm
auto_vec:
        test    ecx, ecx
        jle     .L1
        cmp     ecx, 1
        je      .L3
        lea     rax, [rdi-4]
        mov     r8, rax
        sub     r8, rsi
        cmp     r8, 8
        jbe     .L3
        sub     rax, rdx
        cmp     rax, 8
        jbe     .L3
        lea     eax, [rcx-1]
        mov     r9d, ecx
        cmp     eax, 2
        jbe     .L11
        mov     r8d, ecx
        xor     eax, eax
        shr     r8d, 2
        sal     r8, 4
.L5:
        movdqu  xmm0, XMMWORD PTR [rsi+rax]
        movdqu  xmm2, XMMWORD PTR [rdx+rax]
        paddd   xmm0, xmm2
        movups  XMMWORD PTR [rdi+rax], xmm0
        add     rax, 16
        cmp     r8, rax
        jne     .L5
        mov     eax, ecx
        and     eax, -4
        mov     r8d, eax
        cmp     ecx, eax
        je      .L1
        sub     ecx, eax
        mov     r9d, ecx
        cmp     ecx, 1
        je      .L7
.L4:
        mov     ecx, r8d
        movq    xmm0, QWORD PTR [rsi+rcx*4]
        movq    xmm1, QWORD PTR [rdx+rcx*4]
        paddd   xmm0, xmm1
        movq    QWORD PTR [rdi+rcx*4], xmm0
        test    r9b, 1
        je      .L1
        and     r9d, -2
        add     eax, r9d
.L7:
        cdqe
        mov     edx, DWORD PTR [rdx+rax*4]
        add     edx, DWORD PTR [rsi+rax*4]
        mov     DWORD PTR [rdi+rax*4], edx
        ret
.L3:
        movsx   rcx, ecx
        xor     eax, eax
        sal     rcx, 2
.L9:
        mov     r8d, DWORD PTR [rdx+rax]
        add     r8d, DWORD PTR [rsi+rax]
        mov     DWORD PTR [rdi+rax], r8d
        add     rax, 4
        cmp     rax, rcx
        jne     .L9
.L1:
        ret
.L11:
        xor     r8d, r8d
        xor     eax, eax
        jmp     .L4
```



