- Feature Name: simd-redux
- Start Date: 2017-03-13
- RFC PR:
- Rust Issue:

# Table of contents

* [Summary][summary]
* [Terminology][terminology]
* [Motivation][motivation]
    * [Vendor independent interface][vendor-independent-interface]
    * [Vendor dependent intrinsics][vendor-dependent-intrinsics]
    * [A brief survey][a-brief-survey]
* [Detailed design][design]
    * [Vendor dependent intrinsics][vendor-dependent-intrinsics-1]
    * [Vendor independent interface][vendor-independent-interface-1]
        * [A World Without][a-world-without]
* [How We Teach This][how-we-teach-this]
* [Drawbacks][drawbacks]
* [Alternatives][alternatives]
* [Unresolved questions][unresolved]

# Summary
[summary]: #summary

This RFC proposes the stabilization of explicit support for vendor dependent
intrinsics and a small API for vendor independent SIMD operations. Vendor
dependent intrinsics are normal Rust functions that roughly match the
interfaces defined by various vendors, such as
[Intel's intrinsics](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)
or
[ARM's NEON intrinsics](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0491h/CIHJBEFE.html).

The primary goal of this RFC is to provide explicit access to vendor dependent
intrinsics to programmers on stable Rust. The secondary goal of this RFC is to
start the experimentation of higher level SIMD APIs in the Rust ecosystem. This
is the *beginning* of SIMD on Rust, not the end.

# Terminology
[terminology]: #terminology

Before getting into the meat of the RFC, it would be a good idea to cover some
basic terminology used throughout this RFC.

* **SIMD** - SIMD stands for Single Instruction, Multiple Data. It is an
  umbrella term to refer to a class of CPU instructions that exploit hardware
  level parallelism to operate on multiple piece of data simultaneously.
* **SIMD vector** - Also sometimes written as simply "vector," is the unit
  of data that SIMD CPU instructions act upon. An example of a SIMD vector is
  `f32x4` (or Intel's equivalent, `__m128`), which is a sequence of 4 32-bit
  single-precision floating pointer numbers, occupying exactly 128-bits.
  Vectors come in many shapes and sizes, for example a `u8x16` is a 128-bit
  vector of 16 unsigned 8-bit integers and a `i32x8` is a 256-bit vector of
  8 signed 32-bit integers.
* **lane configuration** - A lane configuration refers to the number of units
  in a SIMD vector. For example, a `f32x4` vector has 4 32-bit lanes, and
  a `i8x32` has 32 8-bit lanes.
* **vendor** - A company that produces CPUs, e.g., Intel and ARM.
* **vendor independent API** - An API defined in terms of SIMD vectors that
  works on all supported platforms and *may* use special SIMD instructions on
  some platforms. For example, adding two vectors of `f32x4` can be done on
  CPUs with Intel SSE2 support in a single CPU instruction. If no such
  analogous support is available, then the addition of two vectors is done
  using normal instructions. The point of a vendor independent API is that it
  *abstracts away* the portability concerns of SIMD. Using a vendor
  independent API does not require any conditional compilation.
* **vendor dependent API** - An API defined in terms of specific instructions
  made available by vendors. Using a vendor dependent API is not portable,
  which means all uses must somehow check that the APIs used are actually
  available on the platform where the program is executed. A vendor dependent
  API is typically defined by the vendor itself using C functions and types,
  however, this RFC proposes its own mechanical translation of the API to Rust.
* **instruction set extensions** - A set of CPU instructions that have been
  added to the set of instructions normally supported by a CPU vendor. For
  example, `x86_64` corresponds to a specific set of CPU instructions that
  are guaranteed to be available on all CPUs advertising `x86_64` support.
  Over the years, new instructions have been added on top of this set as new
  CPUs have been released. This means that the set of instructions supported
  by any given `x86_64` CPU varies depending on the specific model of the CPU.
  **Note that instruction set extensions are mostly operations that act upon
  SIMD vectors.** Even though not all extensions are SIMD, we still sometimes
  use SIMD as an umbrella term to refer to them.
* **SSE** - SSE stands for Streaming SIMD Extensions. SSE is an example of
  a set of instruction set extensions for `x86` and `x86_64`. There are several
  versions of SSE that have been released over the years. Confusingly, `x86_64`
  actually includes SSE2 as part of its base set of instructions, which means
  every single `x86_64` CPU is guaranteed to have all SSE and SSE2
  instructions. Instructions introduced in SSE3, SSSE3, SSE4.1, SSE4.2, etc.,
  are not guaranteed to be available on any given `x86_64` CPU. All intructions
  introduced by the SSE extensions operate on vectors no larger than 128 bits.
* **AVX** - AVX stands for Advanced Vector Extensions. AVX is another class
  of intruction set extensions for `x86_64`. Unlike SSE, the focus of AVX is
  on 256 bit vectors.
* **AVX-512** - The latest set of extensions introduced by Intel for `x86_64`.
  Unlike AVX, AVX-512's focus is on 512-bit vectors.
* **NEON** - NEON is an instruction set extension, just like SSE, except it's
  for ARM CPUs, not Intel.
* **compiler intrinsic** - A special type of function made available by the
  compiler. (**Note**: This RFC does not propose the stabilization of any
  compiler intrinsics.)
* **vendor intrinsic** - A normal Rust function whose API mechanically matches
  the API specified by a vendor.
* **autovectorization** - An automatic process by optimizing compilers to
  generate code which uses SIMD instructions, even if you aren't using a SIMD
  API.

# Motivation
[motivation]: #motivation

## Vendor independent interface
[vendor-independent-interface]: #vendor-independent-interface

A vendor independent interface provides reasonably high-level access to SIMD
vector types and a set of basic operations that can be performed on those
vectors. As an independent interface, users of this API do not need to be
concerned about portability or the details of specific platform support.

A crucial reason why this RFC proposes a small vendor independent interface
is that it would be otherwise very difficult for a crate outside of `std` to
build their own vendor independent interface. (This is discussed in more depth
later.)

## Vendor dependent intrinsics
[vendor-dependent-intrinsics]: #vendor-dependent-intrinsics

To a first approximation, vendor dependent intrinsics provide a *reliable*
way of executing specific CPU instructions. These CPU instructions are often
desirable to use because they typically offer some kind of performance benefit.

The key word to focus on here is "reliable." In today's world, optimizing
compilers will introduce SIMD instructions in your program where applicable.
This process is generally referred to as autovectorization. But, optimizers
are generally opaque, which can make it hard to be certain that you're making
the most effective use of your CPU. Moreover, there are *thousands* of vendor
dependent intrinsics, and trying to convince an optimizing compiler to use a
specific one in any given circumstance may be fruitless. Some intrinsics are so
specialized that it may be very hard for a compiler to use it automatically.

Therefore, the fundamental role of vendor dependent intrinsics is to provide
programmers with a *reliable* way of executing a particular CPU instruction.

At a higher level, the actual use cases for these specialty instructions are
boundless. SIMD intrinsics are used in graphics, multimedia, linear algebra,
text search and more. The ubiquity of vendor dependent intrinsics is so far
reaching that discovering a new algorithm using specific SIMD intrinsics is
generally considered to be a publishable result. Therefore, it is paramount
that a systems language like Rust provide access to a *familiar* API that
provides vendor dependent intrinsics.

## A Brief Survey
[a-brief-survey]: #a-brief-survey

To further motivate this RFC, we should take a brief look to see how the
existing unstable SIMD features in Rust are being used.

* The [`simd` crate](https://github.com/rust-lang-nursery/simd) itself
  obviously makes heavy use of unstable SIMD features to provide a convenient
  high-level cross platform interface to SIMD vectors. More will be said about
  this crate later, but a goal of this RFC is to permit this crate to compile
  on stable Rust while exposing the same API it does today.
* The [`encoding_rs` crate](https://github.com/hsivonen/encoding_rs) uses
  the `simd` crate to assist with speedy decoding. It also makes use of a few
  vendor dependent intrinsics using the existing `platform-intrinsics`
  infrastructure. Unfortunately, it uses a special LLVM intrinsic called
  `simd_shuffle16`, which isn't part of the proposed API in this RFC. Other
  than that, this crate should be compilable with SIMD support on Rust stable
  with the plan in this RFC.
* The [`bytecount` crate](https://github.com/llogiq/bytecount) uses the `simd`
  crate with AVX2 extensions to accelerate counting bytes.
* The [`regex` crate](https://github.com/rust-lang/regex) uses the `simd`
  crate with SSSE3 extensions to accelerate multiple substring search.
  (See also the [`teddy` crate](https://github.com/jneem/teddy) which makes
   use of LLVM intrinsics like `simd_shuffle16` and `simd_shuffle32` directly.)
* The [`bitintr` crate](https://github.com/gnzlbg/bitintr) uses the
  `platform-intrinsic` infrastructure to access `bmi2` intrinsics which should
  be made available through Rust's standard library support for vendor
  dependent intrinsics.

There are other crates in the Rust ecosystem that use SIMD and all of them
share a single crucial bit in common: the SIMD functionality only works on
nightly Rust. There is a clear and pressing desire among Rust programmers to
get the most out of their CPUs!

# Detailed design
[design]: #detailed-design

## Vendor dependent intrinsics
[vendor-dependent-intrinsics-1]: #vendor-dependent-intrinsics-1

## Vendor independent interface
[vendor-independent-interface-1]: #vendor-independent-interface-1

### A World Without
[a-world-without]: #a-world-without

It would possible to trim this RFC down quite a bit by removing the vendor
independent interface. Instead, we'd be left with only a definition of cross
platform SIMD vector types and a host of vendor dependent intrinsics, as
outlined in the previous section. However, if we do this, it will be very hard
for crates in the ecosystem to provide their own vendor independent API on top
of the SIMD vector types exported by `std`.

For example, let's say we decided to punt on the vendor independent API and an
industrious individual wanted to provide a cross platform `Mul` implementation
on only the SIMD type `i32x4`.
[As explained
here](https://internals.rust-lang.org/t/getting-explicit-simd-on-stable-rust/4380/209),
this would be quite difficult. Namely, each platform would need to use its own
dedicated SIMD instruction, and *within* each platform, certain strategies
might be better than others, depending on the instruction set extensions
supported by the CPU. For example, on `x86_64`, if SSE 4.1 is available, then
the `_mm_mullo_epi32` intrinsic can be used, but failing that, some intrinsics
from SSE2 might be combined to get the desired result. And of course, if no
special SIMD intrinsics are available, then this individual would also have to
provide a manual implementation as well.

This then has to be repeated for each SIMD type and for each of the elementary
operations. Lest you think that the process ought to be the same for each SIMD
type, rest assured that some instruction set extensions (like SSE from Intel)
have somewhat inconsistent support for all operations on all permutations of
types.

A testing strategy would need to be employed, and that testing would need to
account for all platforms supported.

The only reason why it's so simple for us to provide this vendor independent
API is because all this work has already been done for us by LLVM.


# How We Teach This
[how-we-teach-this]: #how-we-teach-this

What names and terminology work best for these concepts and why?
How is this idea best presented—as a continuation of existing Rust patterns, or as a wholly new one?

Would the acceptance of this proposal change how Rust is taught to new users at any level?
How should this feature be introduced and taught to existing Rust users?

What additions or changes to the Rust Reference, _The Rust Programming Language_, and/or _Rust by Example_ does it entail?

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
