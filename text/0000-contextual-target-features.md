- Feature Name: contextual_target_features
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Rust's target feature [RFC #2045 initially proposed contextual target features](https://github.com/rust-lang/rfcs/blob/master/text/2045-target-feature.md#conditional-compilation-cfgtarget_feature).

That RFC left the implementation of this behavior unanswered, and in the many years since it was accepted, it has not been possible to implement this behavior.

This RFC extends RFC #2045 with a `#[target_feature(caller)]` attribute and `is_{arch}_feature_enabled!` macro to implement this feature.

# Motivation
[motivation]: #motivation

When using target features, it's common to use conditional compilation to modify code based on the available features, using `cfg`:

```
if cfg!(target_feature = "avx") {
    // something fast with AVX
} else {
    // some default implementation
}
```

RFC #2045 proposed that `cfg` would adjust the `target_feature` value depending on the enabled features of the enclosing function.
However, `cfg` values are now understood to be consistent across an entire crate--when a function is tagged with `#[target_feature(enable = "...")]`, `cfg` does not reflect the enabled features.
The proposed macro `is_{arch}_feature_enabled!` returns a `bool` indicating if the enclosing function supports a feature.

RFC #2045 also proposed that this `cfg` value would respect inlining and evaluate to the enabled features of the resulting function after inlining.
This is ultimately not possible to implement because the last inlining step happens long after constants are evaluated.
The proposed attribute `#[target_feature(caller)]` generates copies of the function with target features matching its caller, hoisting the `is_{arch}_feature_enabled!` computation above any optimizations, while still allowing inlining.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The macro `is_{arch}_feature_enabled!` returns a const bool indicating whether a feature is enabled in the enclosing function. For example:
```
#[target_feature(enable = "avx")]
fn foo() {
    assert!(is_x86_feature_enabled!("avx"))
}
```

The function attribute `#[target_feature(caller)]` creates copies of the function with its caller's target features enabled. For example:
```
#[target_feature(caller)]
fn has_avx() -> bool {
    is_x86_feature_enabled!("avx")
}

fn no_features() {
    assert!(!has_avx())
}

#[target_feature(enable = "avx")]
fn avx_enabled() {
    assert!(has_avx())
}
```

Unlike `#[target_feature(enable = "...")]`, `#[target_feature(caller)]` can always be marked `#[inline(always)]`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The macro `is_{arch}_feature_enabled!` is implemented by an intrinsic.
After monomorphization, the intrinsic is evaluated to a const `bool`.

Calling a function tagged with the attribute `#[target_feature(caller)]` introduces a new monomorphization of the function with enabled features matching the caller.

# Drawbacks
[drawbacks]: #drawbacks

A user could potentially end up with many monomorphizations if they use many unique target feature sets.
However, when using many sets of target features, this pattern is likely to be implemented manually and result in a similar binary size.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Some form of conditional compilation based on target features is highly sought after among user of `#[target_feature]`.
This RFC is designed to be the most minimal implementation necessary to complete the unimplemented features of RFC #2045.
No fundamental new language features are introduced, and the proposed changes are complementary to existing `#[target_feature]` mechanisms, such as runtime detection and `target_feature_11`.

As a real-world example, consider `std::simd`'s [`swizzle_dyn` function](https://github.com/rust-lang/portable-simd/blob/5523a313b503290a2cf93a956f347695e71f09e4/crates/core_simd/src/swizzle_dyn.rs#L17-L106).
This function reorders bytes in SIMD vectors using special instructions available in many SIMD extensions represented by a number of target features.
The current implementation uses `cfg` and has no knowledge of the caller's features.
On x86-64 this is particularly bad, because the base features don't support any SIMD byte reordering instructions!
This function is only usable with the relatively obscure `build-std` feature, and is incompatible with runtime feature detection.
Marking this function with `#[target_feature(caller)]` would allow it to work as intended.

For an example outside the compiler, consider the [`autobahn_hash` crate's `mul_lo_hi` function](https://github.com/calebzulawski/autobahn-hash/blob/f35d18565b996a162d1cfbc18abd268b940f4ced/src/lib.rs#L83-L91).
This function performs a variation of SIMD multiplication as a basic building block of the hash function.
The current implementation uses `cfg` to perform an optimization on AArch64 when SVE is not present.
In this form, the function is not compatible with `#[target_feature(enable = "sve")]`.
If the hash is inlined into a function with SVE, the suboptimal non-SVE optimization is still used.
Marking this function (and the rest of the hash functions in the crate) with `#[target_feature(caller)]` would allow calling the hash function in contexts with SVE enabled.

Both of these example functions are very small and intended to be inlined into their callers, which makes runtime detection not an option.

## RFC #3449
This design is inspired by my previous attempt at solving the same problem, [RFC #3449: Contextual target feature detection](https://github.com/rust-lang/rfcs/pull/3449).

[Some insightful comments](https://github.com/rust-lang/rfcs/pull/3449#issuecomment-1596335701) raised concerns about the unreliability and complexity of relying on the inliner and backend to evaluate `is_{arch}_feature_enabled` so late in the compilation process.

This RFC outlines a simpler and more reliable approach that doesn't rely on inlining, but is still compatible with inlining.

## RFC #3528
[RFC #3528: Struct target features](https://github.com/rust-lang/rfcs/pull/3525) also proposes multiple monomorphizations for target features, by using a struct annotated with the enabled target features.

The main improvement of this RFC over *struct target features* is that this design introduces a substantially simpler API that covers primarily the same use cases.

RFC #3528 supports inheriting target features not from the caller but arbitrarily along the call stack.
While potentially useful in rare circumstances, the vast majority of situations require the entire call stack below a function to have certain features enabled.

On the other hand, this RFC leverages the compiler to inject target features without requiring the caller to construct a marker type and pass it to the function, resulting in less noisy function signatures that are also appropriate for public interfaces.
It might be possible to introduce that capability to RFC #3528, but that further complicates an already complicated API.

RFC #3528 also diverges much more substantially from established `#[target_feature]` expectations.
With the proposed target feature structs, there would be two ways to provide codegen options (types and attributes) and two ways to ensure target feature safety (types and `target_feature_11`).
In comparison, this RFC is an extension to the RFC that established `#[target_feature]`.

# Prior art
[prior-art]: #prior-art

Clang's `target_clones` attribute and Rust's [`multiversion` crate](https://crates.io/crates/multiversion) create copies of functions with different target features, however these only select the called function by runtime detection.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should `#[target_feature(caller)]` polymorphism be implemented with hidden compiler-generated generic parameters, or a new mechanism?

# Future possibilities
[future-possibilities]: #future-possibilities

- A lint could warn when using `cfg` to query target features in a `#[target_feature]` function.
- An extended form of the attribute could limit which features are inherited, potentially reducing the number of monomorphizations.
- Runtime detection macros `is_{arch}_feature_detected!` could query `is_{arch}_feature_enabled!` to avoid unnecessary detection.
- Some form of "compile time `if`" could allow branches to safely call target feature functions using `target_feature_11`
