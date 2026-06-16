+++
date = '2026-06-15T03:12:08+08:00'
draft = false
title = 'Anatomy of an Addition'
+++

{{< katex >}}

## Introduction

When dabbling in the designs of some hash/PRNG/encrypt functions, I often notice a mix of bitwise operations and mod-2<sup>k</sup> additions in the formulas.

And you can see it for yourself, in the code of the PRNG `xoshiro256++`, or within the structure of the `SHA-2` hash family, inside of the encryption algorithm `Salsa20`, and so on...

> [!CAUTION]
> *(NOTE: add the list of algorithm that uses these algorithms)*

{{< highlight c "hl_lines=7" >}}
static uint64_t s[4];

uint64_t next(void) {
    //        rotate bits operation   mod 2^64 addition
    //                       |        |           |
    //                      vvvv      v           v
    const uint64_t result = rotl(s[0] + s[3], 23) + s[0];
    const uint64_t t = s[1] << 17;
    s[2] ^= s[0];
    s[3] ^= s[1];
    s[1] ^= s[2];
    s[0] ^= s[3];
    s[2] ^= t;
    s[3] = rotl(s[3], 45);
    return result;
}
{{< /highlight >}} 
<figcaption> xorshift256+ source code</figcaption>

{{< figure
  src="img/sha2.png"
  default=true
  width=300
  caption="`SHA-2` diagram: XORs, ANDs, ORs, bitshifts and + mod 2<sup>32/64</sup>"
>}}

{{< figure
  src="img/salsa20.png"
  default=true
  width=500
  caption="From Bernstein's Salsa20 [paper](https://cr.yp.to/snuffle/salsafamily-20071225.pdf)"
>}}

I have a few speculations on why cryptographers love this combo:

*Firstly*, `xor` and `add` are built-in instructions in the CPU, so they ought to be fast.

*Secondly*, mixing `xor` with `add` prevents algebraic attacks from happening: XOR is addition in \(\mathbb{F}_2^k\), while ADD is addition in \(\mathbb{Z}/n\mathbb{Z}\). They are not compatible algebraic groups: The carries resulted in \(\mathbb{Z}\)-addition aren't translated well in \(\mathbb{F}_2\) system of equations, because \(\mathbb{F}_2\)-additions don't have carries. In a sense, each `xor` step is non-linear to each `add` step and vice versa, which twarts cryptanalyzers the chances to build a reliable system of linear equations.

These are the assumptions I often have in my mind, so whenever attempting to crack cryptography formulas that involves these interwined ADD-XOR steps, my mind just goes: *"Eh, why bother? It's non-linear anyways. It's not like we can build a formula out of it."*

**But is it actually the case?**

In 2024, the BRICS+ CTF was held, and one cryptography challenge caught my attention, attacking `xoshiro256++`. The requirements were simple: Given 1337 *consecutive* outputs produced by `xoshiro256++`, find the initial values from the list used to encrypt the flag. And I just love the solution the author came up with. However, this post isn't about discussing this particular CTF problem, but about the question I was inspired from the challenge:

**How non-linear adding mod 2<sup>k</sup> really is?**
