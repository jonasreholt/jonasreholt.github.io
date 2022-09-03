---
layout: post
title:  "Native Trampoline"
date:   2022-09-03 16:59:29 +0200
categories: jekyll update
---
# Project Info
* Written in C++
* [Github repository](https://github.com/jonasreholt/NativeTrampoline)

# Introduction
A native trampoline prepares a class function so it can be used as a callback method for
native code i.e. code written in C. This preparation is needed, as non-static class
functions usually has an implicit parameter pointing to the instance object. This is
particularly a problem in C++, as its definition of method pointers doesn't easily
translate to function pointers.

In languages such as C# this preparation is handled by the compiler via the Delegate
type. However, as C++ does not have anything mimicking this at language level, it
must be implemented on top of it.

Don Clugstons article [Member Function Pointers and the Fastest Possible C++ Delegates](https://www.codeproject.com/Articles/7150/Member-Function-Pointers-and-the-Fastest-Possible)
has a good explanation on C++ function and member function pointers. The most important
note for my project for the native trampoline is that member function pointers consists
of multiple fields, and cannot be converted to function pointers; even more annoying,
different compilers implements member function pointers differently.

# Implementation
First off to use the native trampoline, you must create a C handle function that makes
the implicit `this` parameter explicit, i.e. either wrap it in a static function or a
C-like function. This could for a small example look like this:

```C++
class Fish {
private:
    int _classifier;
public:
    Fish(int c) { _classifier = c; }
    void sort() { std::cout << "I'm the fish " << _classifier << std::endl; }
}

void cSort(Fish* instance) { instance->sort() }
```

This needs to be done, as there are no way of obtaining the memory location of the code 
in a non-static member function pointer in C++, due to the implementation choice explained
by Clugston.

One can then prepare the function such that it can be called from within a native language,
using the native trampoline. This is simply done by invoking the static function below:

```C++
auto nativePtr = Trampoline::Jump0Param<Fish>(fish, cSort);
```

As the function name indicates, it has only been implemented for member functions having
0 explicit parameters. This is simply because the line had to be drawn at some point, as
the trampoline can either be made slower or more function should be made to prepare function
pointers with more arguments.

The trampoline then creates a memory block with execution privileges using `mmap`,
fills it with the necessary assembly code, and returns a pointer to this memory block.
X86-64 assembly code is used, as this is what my computer uses. The assembly
code needed for the 0 parameter trampoline is (in AT&T syntax):

```
push   %rbp
mov    %rbp, %rsp
movabs $0x<instance pointer>, %rdi
movabs $0x<function pointer>, %rax
callq  *%rax
pop    %rbp
retq
```

What this does is simply:
1. Save the frame pointer on the stack.
2. Move the instance pointer into `%rdi` which holds the first function argument.
3. Move the function pointer into `%rax` so that a long call can be made to the
   function.
4. Call said function.
5. Move the frame pointer back from the stack into the `%rbp` register.
6. Return from the trampoline function.
   
If one wanted to create a trampoline for a function containing more parameters, they would
have to be added into the appropriate registers before the `callq` instruction. If enough
parameters are needed, the stack would also be needed to pass the arguments.