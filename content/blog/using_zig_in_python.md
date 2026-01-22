---
title: Using Zig in Python
date: 2026-01-21
---

Lets say you have a Python application and want to speed up a piece of code, you would have various options like making Algorithmic improvoments, using better data structures, etc. However another option you would have is to write that piece of code in a low-level language (like C, Zig, Odin etc..) and use it in your application. In this post lets see how we can use Python cffi and Zig to do the same.

---
## Prerequisites

Install cffi
```bash
pip install cffi
```
---

## Minimal Example

Lets start with a small function and get it to work. 

First lets create a Zig file (`math.zig`) with a simple method to add 2 integers.
```zig
export fn add(a: i32, b: i32) i32 {
    return a + b;
}
```
We need to add export so that this `add` function is available for others to use.

Lets compile this and create a dynamic library. (We can then load this from python)

```bash
zig build-lib -dynamic math.zig
```
This should create a libmath.dylib (if you are on MacOS)

Now lets use this in python
```py
import os
from cffi import FFI
ffi = FFI()

ffi.cdef(
    """
    int add(int a, int b);
    """
)

math = ffi.dlopen(os.path.abspath("libmath.dylib"))

print(math.add(2, 3))
```
Here we did a few things
1. Imported `FFI` from `cffi`
2. Define our exported function declaration using `cdef`. As you might have guessed cdef is the c definition of the function we declared in Zig.
3. Open the dylib using `ffi.dlopen`
4. Call our exported function using `math.add`

If you did everything correctly, you should be able to run the basic.py 
```bash
⇛ python3 basic.py
5
```
---

## Levenshtein Distance

To make this more interesting, lets say you are building a Fuzzy finder or auto-suggest in Python and using Levenshtein distance as an algorithm to get closest matches for a given word.

The below is levenshtein distance in python. Its a simple algorithm that uses Dynamic Programming to figure out the number of edits you have to make to convert one string to the other. For example levenshtein("den", "pen") = 1, since you can replace d with p.

```python
def levenshtein(s1, s2):
    if len(s1) == 0:
        return len(s2)
    if len(s2) == 0:
        return len(s1)

    m, n = len(s1), len(s2)
    dp = [[0 for _ in range(n+1)] for _ in range(m+1)]

    for i in range(m+1):
        dp[i][0] = i
    for j in range(n+1):
        dp[0][j] = j

    for i in range(1, m+1):
        for j in range(1, n+1):
            c1, c2 = s1[i-1], s2[j-1]
            if c1 == c2:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])

    return dp[m][n]

print(levenshtein("den", "hen"))
```

Running the above should print `1`
```bash
⇛ python3 lev.py
1
```

Now lets replicate the same in Zig

```zig
const std = @import("std");

export fn levenshtein(s1: [*:0]const u8, s2: [*:0]const u8) u32 {
    const m = std.mem.len(s1);
    const n = std.mem.len(s2);
    if (m == 0) return @intCast(n);
    if (n == 0) return @intCast(m);

    const allocator = std.heap.c_allocator;

    var dp = allocator.alloc([]u32, m + 1) catch return 0;
    defer allocator.free(dp);

    for (dp) |*row| {
        row.* = allocator.alloc(u32, n + 1) catch return 0;
    }
    defer for (dp) |row| {
        allocator.free(row);
    };

    for (0..m+1) |i| {
        for (0..n+1) |j| {
            dp[i][j] = 0;
        }
    }

    for (0..m+1) |i| {
        dp[i][0] = @intCast(i);
    }

    for (0..n+1) |j| {
        dp[0][j] = @intCast(j);
    }

    for (1..m+1) |i| {
        for (1..n+1) |j| {
            if (s1[i-1] == s2[j-1]) {
                dp[i][j] = dp[i-1][j-1];
            } else {
                dp[i][j] = 1 + @min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]);
            }
        }
    }

    return dp[m][n];
}
```

The logic (and most of the code) is exactly the same as Python, but a few things to note here
1. We are accepting the input params as `[*:0]const u8`. This is the equivalent to a C string in Zig
2. We are using `std.heap.c_allocator` to allocate space for the strings

Now lets compile this again like before

```bash
zig build-lib -dynamic lev.zig
```

This should create a `liblev.dylib`

Now lets update our Python file to use this levenshtein library

```py
import os
from cffi import FFI

ffi = FFI()
ffi.cdef(
    """
    unsigned int levenshtein(char* s1, char* s2);
    """
)

lev = ffi.dlopen(os.path.abspath("liblev.dylib"))

def levenshtein(s1, s2):
    if len(s1) == 0:
        return len(s2)
    if len(s2) == 0:
        return len(s1)

    m, n = len(s1), len(s2)
    dp = [[0 for _ in range(n+1)] for _ in range(m+1)]

    for i in range(m+1):
        dp[i][0] = i
    for j in range(n+1):
        dp[0][j] = j

    for i in range(1, m+1):
        for j in range(1, n+1):
            c1, c2 = s1[i-1], s2[j-1]
            if c1 == c2:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])

    return dp[m][n]

print(levenshtein("den", "hen") == lev.levenshtein(b"den", b"hen"))
```

Most of the code is self explanatory, except the fact that when calling lev.levenshtein we are passing Python string objects as bytes so that the low-level implementations can work with them.

Running this should print True
```py
⇛ python3 lev.py
True
```
---

## Comparison
Now lets see if we get any performance benefit by doing this. 

The idea is this
1. We take any random english word
2. Get top 10 matches (from an english dictionary) for the all the prefixes of the word using the Python implementation and Zig Implementation

Why prefixes? Well if you think of auto-suggest, it gets called when you are typing something. For example, you could be typing 'P' ... 'Py' ... 'Pyt' ... and you keep getting updated results

I also added some extra checks to see that the results from both approaches match. Full code can be found [here](https://github.com/sainadhx/py-zig/blob/main/perf_test.py)

```bash
⇛ python3 perf_test.py
Results match!

Python lev:   Min: 0.388583168387413, Max: 26.69854206033051, Total: 155.6180831976235 ms
Zig lev:      Min: 0.6060001906007528, Max: 3.7530839908868074, Total: 25.96958540380001 ms
```

The total time spent in Zig version is about 6x less than the python verion for the word ("Programming") on my machine.
