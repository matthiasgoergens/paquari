+++
title = "Seeing through the template black magic"
date = 2026-09-17T19:30:00+08:00
description = "Boost builds its taus88 random number generator from a tower of nested C++ templates, said to guarantee that the compiler merges everything into optimal code. I check whether plain C functions with constant arguments get the same treatment from GCC and Clang, on x86-64 and RISC-V. They do — with one real caveat."
+++

A [recent Hacker News discussion][hn-thread] about Boost's random number
library featured this typedef:

```cpp
typedef xor_combine_engine<
  xor_combine_engine<
    linear_feedback_shift_engine<uint32_t, 32, 31, 13, 12>, 0,
    linear_feedback_shift_engine<uint32_t, 32, 29, 2, 4>, 0>, 0,
  linear_feedback_shift_engine<uint32_t, 32, 28, 3, 17>, 0> taus88;
```

`taus88` is three linear-feedback-shift registers with different
constants, advanced independently and XORed together. The template tower
instantiates the same step function three times, once per constant set,
and the compiler inlines and merges the copies into one small function.
One commenter called this template-based compile-time programming and
credited it with guaranteeing merged, optimal code *"even in a 1999 C++
compiler"*. I [replied][hn-mine] that plain C functions with constant
arguments ought to get the same treatment from any reasonably smart
compiler. So I ran the experiment.

## The contenders

The C++ side is the real boost (1.92), whose LFSR step looks like this —
for `w == 32` the `wordmask()` degenerates to all-ones:

```cpp
const UIntType b = (((value << q) ^ value) & wordmask()) >> (k - s);
const UIntType mask = (wordmask() << (w - k)) & wordmask();
value = ((value & mask) << s) ^ b;
```

The C side is one `static inline` function taking the same parameters as
the template, computing `k - s` and the mask in its body exactly as the
template does — nothing is precomputed by hand:

```c
static inline uint32_t lfsr_step(uint32_t v, int k, int q, int s)
{
    uint32_t b = ((v << q) ^ v) >> (k - s);   /* w == 32: wordmask == ~0u */
    uint32_t mask = ~0u << (32 - k);          /* top k bits */
    return ((v & mask) << s) ^ b;
}

uint32_t taus88_next(taus88_state *st)
{
    st->s1 = lfsr_step(st->s1, 31, 13, 12);
    st->s2 = lfsr_step(st->s2, 29,  2,  4);
    st->s3 = lfsr_step(st->s3, 28,  3, 17);
    return st->s1 ^ st->s2 ^ st->s3;
}
```

As a baseline for "working out the combination yourself" I also wrote
L'Ecuyer's classic hand-specialised macro version of taus88.

Toolchain: GCC 16.2.1 and Clang 22.1.8 on x86-64 Linux, plus RISC-V
(rv64gc) via `clang --target=riscv64-linux-gnu`. First, a functional
check: seeded through boost's own value constructors with one shared
initial state, all six binaries — boost via g++ and clang++, C via gcc
and clang, the macro version, and a minimal template replica — produce
the same ten outputs (first value 2450055554).

One methodological note: to compare code generation you do not need full
binaries. LTO runs at link time, so disassembling the plain object files
with `objdump -d` shows exactly what each compiler emitted for each
translation unit. (The one exception is the LTO experiment below, where
the link *is* the point.)

## x86-64 at -O2: spot the difference

Side-by-side disassembly of `taus88_next`, Godbolt style: boost C++
templates on the left, plain C on the right, GCC 16 -O2. Rows marked
`|`, `<`, `>` differ:

```
taus88_next:                            taus88_next:
        movl    (%rdi), %edx         |          movl    (%rdi), %eax
                                     >          movl    4(%rdi), %edx
        movl    8(%rdi), %esi                   movl    8(%rdi), %esi
        movl    %edx, %eax           |          movl    %eax, %ecx
        sall    $13, %eax            |          sall    $13, %ecx
        xorl    %edx, %eax           |          xorl    %eax, %ecx
        sall    $12, %edx            |          sall    $12, %eax
        andl    $-8192, %edx         |          andl    $-8192, %eax
        shrl    $19, %eax            |          shrl    $19, %ecx
                                     >          xorl    %eax, %ecx
                                     >          leal    0(,%rdx,4), %eax
        xorl    %edx, %eax                      xorl    %edx, %eax
        movl    4(%rdi), %edx        <
        movl    %eax, (%rdi)         <
        leal    0(,%rdx,4), %ecx     <
        xorl    %edx, %ecx           <
        sall    $4, %edx                        sall    $4, %edx
                                     >          movl    %ecx, (%rdi)
        andl    $-128, %edx                     andl    $-128, %edx
        shrl    $25, %ecx            |          shrl    $25, %eax
        xorl    %edx, %ecx           |          xorl    %edx, %eax
        leal    0(,%rsi,8), %edx                leal    0(,%rsi,8), %edx
        xorl    %esi, %edx                      xorl    %esi, %edx
        sall    $17, %esi                       sall    $17, %esi
                                     >          movl    %eax, 4(%rdi)
        xorl    %ecx, %eax                      xorl    %ecx, %eax
        movl    %ecx, 4(%rdi)        <
        shrl    $11, %edx                       shrl    $11, %edx
        andl    $-2097152, %esi                 andl    $-2097152, %esi
        xorl    %esi, %edx                      xorl    %esi, %edx
        movl    %edx, 8(%rdi)                   movl    %edx, 8(%rdi)
        xorl    %edx, %eax                      xorl    %edx, %eax
        ret                                     ret
```

Twenty-eight instructions on each side, with the same operations and the
same constants; the differences are register allocation and scheduling.
Both sides even get the same mask-through-shift rewrite:
`((v & 0xfffffffe) << 12)` becomes `(v << 12) & 0xffffe000` — that is
the `andl $-8192`. The hand-written macro version produces the same code
again. Clang 22 also emits 28 instructions for both versions, with
matching operations and constants, again differing only in register
allocation and scheduling.

## RISC-V at -O2: no difference to spot

Cross-compiling with Clang for rv64gc, the C version and the template
version emit identical instruction sequences — 28 instructions, same
registers, same order:

```
        lw      a1, 0(a0)
        lw      a2, 4(a0)
        lw      a3, 8(a0)
        slli    a4, a1, 13
        xor     a4, a4, a1
        andi    a1, a1, -2
        slliw   a1, a1, 12
        srliw   a4, a4, 19
        or      a4, a4, a1
        slli    a1, a2, 2
        xor     a1, a1, a2
        slliw   a2, a2, 4
        andi    a2, a2, -128
        srliw   a1, a1, 25
        or      a2, a2, a1
        slli    a1, a3, 3
        xor     a1, a1, a3
        andi    a3, a3, -16
        slliw   a3, a3, 17
        srliw   a1, a1, 11
        or      a3, a3, a1
        xor     a1, a2, a4
        xor     a1, a1, a3
        sw      a4, 0(a0)
        sw      a2, 4(a0)
        sw      a3, 8(a0)
        mv      a0, a1
        ret
```

One qualification: cross-compiling the real boost headers needs a
RISC-V C++ standard library, which this machine lacks, so the RISC-V
C++ side is the self-contained template replica. On x86-64 the replica
compiles to assembly identical to real boost (GCC), which is why I
consider it a fair stand-in here.

## -O3 and -O1

-O3 changes nothing for this function: GCC's -O3 output is identical to
its -O2 output, and Clang's -O3 differs from its -O2 only in register
allocation. Going the other way, -O1 already suffices: both compilers
fully inline and constant-fold the C version there.

## The -O0 control

With optimisation off, neither language gives you the merged function.
The C version keeps three real calls, passing the constants in argument
registers — and inside `lfsr_step` the shifts are variable:

```
        movl    $12, %ecx
        movl    $13, %edx
        movl    $31, %esi
        movl    %eax, %edi
        call    lfsr_step
...
        sall    %cl, %edx        # lfsr_step body: shift by register
        shrl    %cl, %edx
        sall    %cl, %edx
```

The C++ version at -O0 keeps a call tree five levels deep:
`taus88_next` calls the outer `xor_combine_engine::operator()`, which
calls the inner one, which calls each
`linear_feedback_shift_engine::operator()`, and those even make a real
call to a `wordmask()` function. But inside each instantiation the
constants are immediates:

```
        call    _ZN5boost...28ELi3ELi17EE8wordmaskEv
        andl    %ebx, %eax
        shrl    $11, %eax
        ...
        sall    $17, %eax
        andl    $-2097152, %eax
```

So for these modern compilers, "even in a 1999 C++ compiler" is half
right: template specialisations keep their constants at any optimisation
level, but merging the instances into one tight function is the
optimiser's work in both languages. What an actual 1999 compiler did, I
leave to anyone with a working copy of one.

## When the compiler can't see the body

These boost templates are defined in headers, so every translation unit
that uses them can see their bodies. Plain C needs the same arrangement:
the step function in a header — a plain `static` function suffices (that
is the pre-C99 idiom; `static inline` just silences `-Wunused-function`
in translation units that include the header without calling it) — or
link-time optimisation. When I moved `lfsr_step` into its own `.c` file,
GCC -O2
still passed the constants in argument registers, but the separately
compiled callee could not fold them: three calls remained, and the
shifts stayed variable (`sall %cl`). Compiling both files with `-flto`
and linking restores the identical 28-instruction, call-free body.

Note this is the mechanism, not a guarantee: inlining and constant
propagation remain heuristics, and a sufficiently large or
sufficiently strange function will defeat them in either language. What
I can claim is this function, these compilers, these flags.

## Verdict

For this function, on GCC 16 and Clang 22, the claim holds: with the
function body visible at the call site and optimisation switched on
(-O1 is enough), a C compiler produces essentially the same machine
code from plain functions as it does from the template tower, and on
RISC-V literally the same machine code. The template version showed no
code-generation advantage in these tests. What it does buy is that the
constants are checked and baked into each instantiation at compile time
— in C, nothing stops a caller from passing run-time variables and
silently losing the folded code — and that specialisation happens even
at -O0.

## Reproduce

```sh
# x86-64, boost C++ vs plain C (boost 1.92, GCC 16 / Clang 22)
g++ -O2 -c taus88_boost.cpp -o boost.o
gcc -O2 -c taus88.c -o c.o
objdump -d --no-show-raw-insn --disassemble=taus88_next boost.o c.o

# RISC-V rv64gc (template replica stand-in, see above)
clang   --target=riscv64-linux-gnu -march=rv64gc -O2 -S core_c.c
clang++ --target=riscv64-linux-gnu -march=rv64gc -O2 -S core_cpp.cpp

# separate translation units: 3 calls remain without, gone with LTO
gcc -O2 -S taus88_tu.c
gcc -O2 -flto -c taus88_tu.c lfsr_step_tu.c
gcc -O2 -flto -shared taus88_tu.o lfsr_step_tu.o -o combined.so
objdump -d --no-show-raw-insn combined.so
```

[hn-thread]: https://news.ycombinator.com/item?id=49735445
[hn-mine]: https://news.ycombinator.com/item?id=49739076
