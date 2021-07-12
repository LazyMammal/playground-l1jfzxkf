# Using Pragma For Compile Optimization

* Adjust compile settings without access to command line.
* Not all command line compile flags can be replicated.

## Configure 

* Ideally, set `march=native` in pragma but this does not work.
* Use instruction targets for "haswell" or "core-avx2".

The bare minimum:
```C++
#pragma GCC optimize("O3,inline")
#pragma GCC target("bmi,bmi2,lzcnt,popcnt")
```

I would personally recommend adding SIMD as well.  The compiler can use it even if you don't code the instructions yourself:
```C++
#pragma GCC target("avx,avx2,f16c,fma,sse3,ssse3,sse4.1,sse4.2") // SIMD
```

Pick whichever you prefer but at least "O3,inline" to get close to native:
```C++
#pragma GCC optimize("O3,inline")             // "inline" won't happen without it
#pragma GCC optimize("O3,fast-math,inline")   // "fast-math" helps auto-vectorize loops
#pragma GCC optimize("Ofast,inline")          // "Ofast" = "O3,fast-math,allow-store-data-races,no-protect-parens"
```

Finally, hint to GLIBCXX to run faster:
```C++
#undef _GLIBCXX_DEBUG // disable run-time bound checking, etc
```

Full example:
```C++
#undef _GLIBCXX_DEBUG                // disable run-time bound checking, etc
#pragma GCC optimize("Ofast,inline") // Ofast = O3,fast-math,allow-store-data-races,no-protect-parens

#pragma GCC target("bmi,bmi2,lzcnt,popcnt")                      // bit manipulation
#pragma GCC target("movbe")                                      // byte swap
#pragma GCC target("aes,pclmul,rdrnd")                           // encryption
#pragma GCC target("avx,avx2,f16c,fma,sse3,ssse3,sse4.1,sse4.2") // SIMD

// Caution! Include headers *after* compile options.
#include <iostream> 
#include <string>
#include <vector>
#include <algorithm>

using namespace std;
```

## Verify

* How to detect if options are working?
* CodinGame does not allow inspection of compiled binary.
* Compiler Explorer at [godbolt.org](https://godbolt.org/)
* Compile settings basically the same as CodinGame (notice the lack of `march=native`):
```
gcc-10 -g -std=c++17 -lm -lpthread -ldl -lcrypt
```

### #pragma GCC target("lzcnt")

Try to compile this snippet (with and without pragma):
```C++
#pragma GCC optimize("O1")
// #pragma GCC target("lzcnt")

int leading_zeros(unsigned x) // count leading 0-bits
{
    return __builtin_clz(x); // LZCNT
}
```

We expect to see `LZCNT` in the assembly instructions:
```
leading_zeros(unsigned int):
        xor     eax, eax
        lzcnt   eax, edi
        ret
```

It should NOT look like this:
```
leading_zeros(unsigned int):
        bsr     eax, edi
        xor     eax, 31
        ret
```

### #pragma GCC optimize("inline")

Snippet:
```C++
#pragma GCC optimize("O1")
// #pragma GCC optimize("inline")

int foo() { return 12; }
int bar() { return foo(); }
```

Expect:
```
foo():
        mov     eax, 12
        ret
bar():
        mov     eax, 12
        ret
```

Failure:
```
foo():
        mov     eax, 12
        ret
bar():
        call    foo()
        ret
```

## Compiling at home

* You may want to compile offline using same source file.
* Pragma options for Haswell may be wrong for your home computer.
* Be sure to set `march=native` on command line.

## Detect march=native

* GCC will use generic CPU type if not set.
* Generic CPU machine lacks many x86 instructions (POPCNT, etc).
* Test for POPCNT to detect if `march=native` is already set.

```C++
#ifndef __POPCNT__

// compile options so CodinGame can behave more like native compile

#endif
```

### More code examples

* LZCNT, TZCNT, POPCNT
* SIMD intrinsics
* Bonus example for Bitboards
* https://godbolt.org/z/Whx3v5aeG
