---
title: GUESS THAT OPTIMIZATION
date: 2026-06-09
draft: false
---
# Guess That Optimization, Struct Initialization
![Guess That Optimization](guess-optimization.png)

It is legend that many hardware engineers working in low level languages like C optimize code more based on "feeling" and what they think the compiler will do. They write bad "optimized" code that's both harder to read and less efficient.

As a hardware engineer myself with little experience as to how compilers actually work and 10+ years of experience coding in C, I fit the bill perfectly.

We will also play today's game with two other contestants. A fellow C developer, and Claude.

All will be put to the test in today's game of Guess That Optimization. Feel free to compete and follow along yourself.

## Introduction To Why
Though the beforehand legend may be overexaggerated, it's not something I am new to, and I, and many others, have fallen victim to many times.

I have come to avoid so many "anti-patterns", bad coding practices, in C, that I really just imagined existing based on assumptions of how the compiler worked internally, and wrote C code to try and baby the compiler into generating clean assembly.

As time has gone on though, I have come to realize, my "babying" of the compiler was just confusing it, making my code harder to optimize, and all at the sacrifice of the reader. Maybe we just deep down think optimized code should look unreadable and hard to follow or something.

Even then, after tonight's game, I came to realize I was still dramatically wrong and should be much more humble about what the compiler will do.

## Rules of the Game
Tonight's game will be run using 3 simple C (technically C++) functions to construct a struct. All C functions behave the same, but use slightly different methods. No fancy math, vectorization, or anything, we're just copying some data. This should be easy, right?

Two compilers will compete tonight to optimize them. GCC 16.1.0 and Clang 22.1.0.
Both of them will be compiling for RISC-V RV32GC instruction set architecture.

I chose RISC-V because it's an assembly language I know the best, and provides clean and simple assembly with hopefully fewer footguns.

Tonight's game was made possible with [Compiler Explorer](https://godbolt.org/), an interactive website to quickly view the assembly output of a compiler.

## The Code
We will be constructing a struct in each function. Each function will operate the same. The struct contains both text, and a 16 bit unsigned value.
```c
#include <stdint.h>

#include <stdbool.h>

#include <string.h>

  

typedef struct {

char name[16];

uint16_t number_of_cats;

} super_cool_thing_t;

  

super_cool_thing_t get_thing1(const char* name, bool i_like_dogs) {

if(i_like_dogs) {

return super_cool_thing_t { // The following appears to be a C++ thing:

name: {'B', 'o', 'b', '\0'},

number_of_cats: 1

};

} else {

return super_cool_thing_t {

name: {'T', 'i', 'm', '\0'},

number_of_cats: 32

};

}

}

  

void get_thing2(super_cool_thing_t* r, const char* name, bool i_like_dogs) {

memset(r, 0, sizeof(*r));

  

if(i_like_dogs) {

r->name[0] = 'B';

r->name[1] = 'o';

r->name[2] = 'b';

r->name[3] = '\0';

r->number_of_cats = 1;

} else {

r->name[0] = 'T';

r->name[1] = 'i';

r->name[2] = 'm';

r->name[3] = '\0';

r->number_of_cats = 32;

}

}

  

super_cool_thing_t get_thing3(const char* name, bool i_like_dogs) {

super_cool_thing_t t = {0};

if (i_like_dogs) {

strcpy(t.name, "Bob");

t.number_of_cats = 1;

} else {

strcpy(t.name, "Tim");

t.number_of_cats = 32;

}

return t;

}
```

Each function returns type `super_cool_thing_t`. If you like dogs, it puts in your name as Bob and ensures you only have 1 cat. If you don't like dogs, it assumes you're Tim, love cats, and gives you 32 cats to go along with it.
### get_thing1
This is the first method that comes to mind for me. I just return 1 of two structs depending on if you like dogs or not.

It's worth noting the `name: value` only compiles if told to be treated as C++, not C. I included this, because I started doing it because of Rust and SystemVerilog. In both, I like it when setting up structs and ports respectively because it's more clear to me what you're setting.

This one is probably the most readable to me, except for my weird way of setting the strings.

### get_thing2
This was my attempt at an "optimized" version. Passing `super_cool_thing_t* r` as a pointer instead of just returning it is probably pointless here. If you don't do that the compiler should just have the caller make space, and pass a pointer to it just like I did here manually.

Anyhow, this to me is equivalent to get_thing1. I just distrusted the compiler and did it all manually.

### get_thing3
This one is a cross between **get_thing1** and **get_thing2**.

Instead of distrusting the compiler with return value optimizations (RVO), I just went ahead and returned the struct.

But instead of having two different struct initializations, I explicitly cleared it, then filled out the structs with strcpy.

## The Contestants' Answers
### Myself
I thought **get_thing2** would be the fastest, then **get_thing1**, and then **get_thing3**.

My reasoning was **get_thing2** was the most direct in operation. It says, everything will be zeros. Then says what each byte should be. To me, that everything is pretty much known easily, just set up the data.

**get_thing1** should be no different to me. I have no reason to distrust RVO right?

Then **get_thing3** just seemed like the worst of both worlds. We're trusting the compiler with RVO and inlining strcpy. So it just seems the most prone to issues.

### Fellow C Developer (awewsomegamer)
Awewsomegamer speculated **get_thing1** would be the fastest. **get_thing2** would come in second. And **get_thing3** being prone to be the slowest.

Very similar reasoning, **get_thing1** was very upfront and direct with the compiler. Set up the caller's call stack with this, that's it, easy.

Then **get_thing2** was speculated to be the same, and should give if not the same output.

Same story with **get_thing3**, it's really just a more indirect version of the other two and we're depending on strcpy to play nicely.

### Claude
Claude rated them as follows.
`get_thing1` — likely slowest or middle
`get_thing2` — likely fastest
`get_thing3` — likely middle or slowest

It justified **get_thing1** as slowest because it distrusted RVO. It stated that it would have to construct two ``super_cool_thing_t`` then likely memcpy the result.

**get_thing2** was seen as the fastest for the same reason. It thought the lack of hidden-pointer ABI overhead on the return value would make it "minimal, predictable, no copying." It also noted the **memset** would likely be optimized out.

**get_thing3** it predicted to be the slowest. It noted the `{0}` zero-init is likely the same as memset, but noted that the strcpy would likely not be optimized out, and act as an unnecessary function call.


## Round 1, Clang
The original function and the generated assembly are provided. I recommend reading through each to see how the compiler approached the problem.
### get_thing1
```c
super_cool_thing_t get_thing1(const char* name, bool i_like_dogs) {

if(i_like_dogs) {

return super_cool_thing_t {

name: {'B', 'o', 'b', '\0'},

number_of_cats: 1

};

} else {

return super_cool_thing_t {

name: {'T', 'i', 'm', '\0'},

number_of_cats: 32

};

}

}
```

```asm
get_thing1(char const*, bool):

addi sp, sp, -16

sw ra, 12(sp)

sw s0, 8(sp)

sw s1, 4(sp)

mv s0, a0

addi a0, a0, 3

bnez a2, .LBB0_2

li a1, 84

li a2, 105

li a3, 109

li s1, 32

j .LBB0_3

.LBB0_2:

li a1, 66

li a2, 111

li a3, 98

li s1, 1

.LBB0_3:

sb a1, 0(s0)

sb a2, 1(s0)

sb a3, 2(s0)

li a2, 13

li a1, 0

call memset

sh s1, 16(s0)

lw ra, 12(sp)

lw s0, 8(sp)

lw s1, 4(sp)

addi sp, sp, 16

ret
```

This operation makes an entire stack frame, calls memset, and uses 23-24 instructions.

Clang's choice to call **memset** is quite weird here, and we will have to investigate more later.

Clang chose to construct the string 1 byte at a time.

### get_thing2
```c
void get_thing2(super_cool_thing_t* r, const char* name, bool i_like_dogs) {

memset(r, 0, sizeof(*r));

  

if(i_like_dogs) {

r->name[0] = 'B';

r->name[1] = 'o';

r->name[2] = 'b';

r->name[3] = '\0';

r->number_of_cats = 1;

} else {

r->name[0] = 'T';

r->name[1] = 'i';

r->name[2] = 'm';

r->name[3] = '\0';

r->number_of_cats = 32;

}

}
```

```asm
get_thing2(super_cool_thing_t*, char const*, bool):

sh zero, 12(a0)

sh zero, 14(a0)

sh zero, 4(a0)

sh zero, 6(a0)

sh zero, 8(a0)

sh zero, 10(a0)

beqz a2, .LBB1_2

li a1, 1

li a2, 98

li a3, 111

li a4, 66

sb a4, 0(a0)

sb a3, 1(a0)

sb a2, 2(a0)

sb zero, 3(a0)

sh a1, 16(a0)

ret

.LBB1_2:

li a1, 32

li a2, 109

li a3, 105

li a4, 84

sb a4, 0(a0)

sb a3, 1(a0)

sb a2, 2(a0)

sb zero, 3(a0)

sh a1, 16(a0)

ret
```

This uses 18 instructions, and no stack frame due to no more **memset** calls.

It's weird to see here our explicit **memset** got inlined, while Clang's implicit **memset** did not. The ordering of the function too is quite different.

Clang chose to clear it two bytes at a time, construct the string one byte at a time, and in general is much nicer than the first.

### get_thing3
```c
super_cool_thing_t get_thing3(const char* name, bool i_like_dogs) {

super_cool_thing_t t = {0};

if (i_like_dogs) {

strcpy(t.name, "Bob");

t.number_of_cats = 1;

} else {

strcpy(t.name, "Tim");

t.number_of_cats = 32;

}

return t;

}
```

```asm
get_thing3(char const*, bool):

sh zero, 12(a0)

sh zero, 14(a0)

sh zero, 4(a0)

sh zero, 6(a0)

sh zero, 8(a0)

sh zero, 10(a0)

bnez a2, .LBB2_2

lui a2, 1751

addi a2, a2, -1708

li a1, 32

srli a3, a2, 16

sh a2, 0(a0)

sh a3, 2(a0)

sh a1, 16(a0)

ret

.LBB2_2:

lui a2, 1575

addi a2, a2, -190

li a1, 1

srli a3, a2, 16

sh a2, 0(a0)

sh a3, 2(a0)

sh a1, 16(a0)

ret
```

This uses 16 instructions, and is the shortest so far.

Both the **memset** and both **strcpy**s got inlined and resulted in an overall shorter output.

Clang both cleared it out two bytes at a time, and constructed the string two bytes at a time resulting in the shorter output.

For clang, **get_thing3** has been the best, while it dropped the ball on **get_thing1.**

## Round 2, GCC
### get_thing1
```c
super_cool_thing_t get_thing1(const char* name, bool i_like_dogs) {

if(i_like_dogs) {

return super_cool_thing_t {

name: {'B', 'o', 'b', '\0'},

number_of_cats: 1

};

} else {

return super_cool_thing_t {

name: {'T', 'i', 'm', '\0'},

number_of_cats: 32

};

}

}
```

```asm
get_thing1(char const*, bool):

li a5,28672

beq a2,zero,.L3

addi a5,a5,-190

li a3,98

li a4,1

.L2:

sb zero,3(a0)

sb zero,4(a0)

sb zero,5(a0)

sb zero,6(a0)

sb zero,7(a0)

sb zero,8(a0)

sb zero,9(a0)

sb zero,10(a0)

sb zero,11(a0)

sb zero,12(a0)

sb zero,13(a0)

sb zero,14(a0)

sb zero,15(a0)

sh a5,0(a0)

sb a3,2(a0)

sh a4,16(a0)

ret

.L3:

addi a5,a5,-1708

li a3,109

li a4,32

j .L2
```

This uses 21-22 instructions and did not produce a whole stack frame with **memset** call like Clang.

GCC produced a much cleaner output than Clang here. It chose to clear it out and construct the string 1 byte at a time.

### get_thing2
```c
void get_thing2(super_cool_thing_t* r, const char* name, bool i_like_dogs) {

memset(r, 0, sizeof(*r));

  

if(i_like_dogs) {

r->name[0] = 'B';

r->name[1] = 'o';

r->name[2] = 'b';

r->name[3] = '\0';

r->number_of_cats = 1;

} else {

r->name[0] = 'T';

r->name[1] = 'i';

r->name[2] = 'm';

r->name[3] = '\0';

r->number_of_cats = 32;

}

}
```

```asm
get_thing2(super_cool_thing_t*, char const*, bool):

sb zero,3(a0)

sb zero,4(a0)

sb zero,5(a0)

sb zero,6(a0)

sb zero,7(a0)

sb zero,8(a0)

sb zero,9(a0)

sb zero,10(a0)

sb zero,11(a0)

sb zero,12(a0)

sb zero,13(a0)

sb zero,14(a0)

sb zero,15(a0)

sb zero,16(a0)

sb zero,17(a0)

li a5,28672

beq a2,zero,.L7

addi a5,a5,-190

li a3,98

li a4,1

sh a5,0(a0)

sb a3,2(a0)

sh a4,16(a0)

ret

.L7:

addi a5,a5,-1708

li a3,109

li a4,32

sh a5,0(a0)

sb a3,2(a0)

sh a4,16(a0)

ret
```

This uses 23 instructions and operates about the same as **get_thing1** when compiled with GCC.

This output is similar to what Clang produced, but with more instructions.

### get_thing3
```c
super_cool_thing_t get_thing3(const char* name, bool i_like_dogs) {

super_cool_thing_t t = {0};

if (i_like_dogs) {

strcpy(t.name, "Bob");

t.number_of_cats = 1;

} else {

strcpy(t.name, "Tim");

t.number_of_cats = 32;

}

return t;

}
```

```asm
get_thing3(char const*, bool):

sh zero,0(a0)

sh zero,2(a0)

sh zero,4(a0)

sh zero,6(a0)

sh zero,8(a0)

sh zero,10(a0)

sh zero,12(a0)

sh zero,14(a0)

sh zero,16(a0)

li a5,28672

beq a2,zero,.L9

addi a5,a5,-190

li a4,98

sh a5,0(a0)

li a5,1

sh a4,2(a0)

sh a5,16(a0)

ret

.L9:

addi a5,a5,-1708

li a4,109

sh a5,0(a0)

li a5,32

sh a4,2(a0)

sh a5,16(a0)

ret
```

This uses 18 instructions, unlike the Clang version, it zeros it out just before overwriting it, leading to fewer instructions, but in general they are very similar.

Here it chose to do everything two bytes at a time just like Clang.

## Results
| Version      | Compiler | Instructions   | Stack Frame | memset call  | Zero strategy     | String strategy   |
| ------------ | -------- | -------------- | ----------- | ------------ | ----------------- | ----------------- |
| `get_thing1` | Clang    | 23–24 + memset | Yes         | Yes          | —                 | 1 byte at a time  |
| `get_thing2` | Clang    | 18             | No          | No (inlined) | 2 bytes at a time | 1 byte at a time  |
| `get_thing3` | Clang    | 16             | No          | No (inlined) | 2 bytes at a time | 2 bytes at a time |
| `get_thing1` | GCC      | 21–22          | No          | No           | 1 byte at a time  | 1 byte at a time  |
| `get_thing2` | GCC      | 23             | No          | No (inlined) | 1 byte at a time  | 2 bytes at a time |
| `get_thing3` | GCC      | 18             | No          | No (inlined) | 2 bytes at a time | 2 bytes at a time |

None of the three contestants were correct!

Even without direct performance metrics here, **get_thing3** seems pretty undoubtedly superior to the others.

On both compilers, it was the only version that both efficiently zeroed the struct and the string.

We also saw no issue with RVO, or inlining. Our explicit **memset**s and **memcpy**s were always inlined, with the only failure occurring on the **memset** Clang added itself.

In general this is a big win for compiler optimizations. This does not leave the results without optimization though, like instead of zeroing the struct two bytes at a time, you could have used `sw` to zero them four bytes at a time. C/C++ as far as I know guarantees structs to be 4 byte aligned. I am not sure why both GCC and Clang avoided this optimization.

We also did not look into the branch optimizations in these functions. Speculative execution may prefer some of these implementations over some of the others; these optimizations would be best explored with actual performance testing on a chosen target platform.

### What can be learned
Well I guess so. I think the lesson here is twofold. Avoid making micro-optimizations about what you think the compiler will do. When you do need to optimize, closely follow the actual compiler output and test on real hardware as opposed to playing a guessing game.

From this experience in isolation, it also appears the compiler benefits from you being direct with what you want to do, not from babying each operation required to do it.

## About That Weird Memset
If you're like me, you're wondering what had to go wrong inside of Clang to lead to that weird **memset** in **get_thing1**? Why would Clang add its own fixed **memset**, and then fail to inline it?

Well I reached out to the LLVM community, and I will do my best to interpret what exactly happened here.

The first thing they noted is that at some point the compiler will see too many zero sets, so instead of inlining it, it will just call **memset**, but why here, it's only clearing 13 bytes?

Well as a test, apparently you can have it clear 12 bytes, instead of 13, and the issue goes away.

```c
super_cool_thing_t get_thing1(const char* name, bool i_like_dogs) {

if(i_like_dogs) {

return super_cool_thing_t {

name: {'B', 'o', 'b', 'y', '\0'},

number_of_cats: 1

};

} else {

return super_cool_thing_t {

name: {'T', 'i', 'm', 'y', '\0'},

number_of_cats: 32

};

}

}
```

```asm
get_thing1(char const*, bool):

bnez a2, .LBB0_2

li a2, 84

li a3, 105

li a4, 109

li a1, 32

j .LBB0_3

.LBB0_2:

li a2, 66

li a3, 111

li a4, 98

li a1, 1

.LBB0_3:

li a5, 121

sb a2, 0(a0)

sb a3, 1(a0)

sb a4, 2(a0)

sb a5, 3(a0)

sh zero, 4(a0)

sh zero, 6(a0)

sh zero, 8(a0)

sh zero, 10(a0)

sh zero, 12(a0)

sh zero, 14(a0)

sh a1, 16(a0)

ret
```

This now looks a lot like what we got from GCC.

But I quickly found it seems to be due to it being an odd number of bytes. Where even numbers will down level to just store halfs while odd numbers end up invoking memset. Very interesting.
