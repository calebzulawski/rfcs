- Feature Name: generic_target_features
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Treat target features as generics, allowing functions to be parameterized by their target features and adjust their implementation accordingly.
Functions accepting generic target features are annotated with `#[target_feature(caller)]`, which monomorphise to a particular set of target features matching the caller.
Additionally, provide `is_{arch}_feature_enabled` macros for querying those generic target features.

# Motivation
[motivation]: #motivation

When using target features, it's common to provide multiple implementations depending on the features available.
This can become cumbersome:

```rust
#[cfg(target_arch = "x86_64")]
#[target_features(enable = "avx2")]
fn foo_avx2() {
    ....
}

#[cfg(target_arch = "x86_64")]
#[target_features(enable = "sse4.1")]
fn foo_sse41() {
    ....
}

#[cfg(target_arch = "aarch64")]
#[target_features(enable = "sve")]
fn foo_sve() {
    ...
}

fn foo_default() {
    ...
}
```

To minimize this, we can conditionally adjust the implementation with `cfg`:

```rust
#[target_feature(enable = "avx2")]
fn caller() {
    foo()
}

fn foo() {
    if cfg!(all(target_arch = "x86_64", target_feature = "avx2)) {
        /* this avx2 code is never called! */
    } else {
        /* this slower code is called instead! */
    }
}
```

This works great, except for one problem--this function is not aware of the `#[target_feature]` attribute.
When this function is called from a function annotated with `#[target_feature]`, only the base target features are used.

We can try using a generic:

```rust
#[target_feature(enable = "avx2")]
fn caller() {
    foo::<true>()
}

fn foo<const AVX2: bool>() {
    if AVX2 {
        /* this code is now called */
    } else {
        /* rather than this slow code */
    }
}
```

Great! The AVX2 implementation is now called.
Unfortunately, this exposes an implementation detail of `foo` as a generic parameter and requires the caller to set it correctly.
Even worse, this is a performance pitfall.
If `foo` is not inlined, it will be compiled with the base target features, rather than the desired `#[target_feature(enable = "avx2")]`.
This will prevent proper code generation and very likely end up even slower than not using target features at all.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC introduces a new function attribute: `#[target_feature(caller)]`.
This attribute acts like `#[target_feature(enable = ...)]`, but inherits its features from its caller.

Additionally, this RFC introduces a new macro: `is_{arch}_feature_enabled!()`.
This macro acts like `cfg!(target_feature = ...)`, but interacts with `#[target_feature]` as well as default features available to `cfg`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Consider the following example:

```rust
#[target_feature(enable = "avx2")]
fn caller_avx2() {
    callee()
}

#[target_feature(enable = "sse4.1")]
fn caller_sse41() {
    callee()
}

#[target_feature(caller)]
fn callee() {
    if is_x86_feature_enabled!("avx") {
        ...
    } else if is_x86_feature_enabled!("sse4.1") {
        ...
    } else {
        ...
    }
}
```

The compiler expands this to something like:

```rust
#[target_feature(enable = "avx2")]
fn caller_avx2() {
    callee::<RustcTargetFeaturesType::AVX2>(),
}

#[target_feature(enable = "sse4.1")
fn caller_sse41() {
    callee::<RustcTargetFeaturesType::SSE41>(),
}

#[target_feature(enable = /* "avx2", "sse4.1", etc. depending on TARGET_FEATURES */)]
fn callee<const TARGET_FEATURES: RustcTargetFeaturesType>() {
    if TARGET_FEATURES.avx2 {
        ...
    } else if TARGET_FEATURES.sse41 {
        ...
    } else {
        ...
    }
}
```

The special `#[target_feature(caller)]` attribute adds a generic parameter to the function corresponding to the caller's target features.
When monomorphized, the appropriate `#[target_feature(enable = "...")]` is added, matching the caller's target features provided in the generic parameter.
Additionally, the `is_target_feature_enabled` macro can query that generic parameter.

Nested instances of `#[target_feature(caller)]` pass the initial caller's features to all callees.

When called from non-`#[target_feature]` functions, the generic parameter is set to the default target features (set by the target and `-Ctarget-feature`).
When used in a non-`#[target_feature]` function, `is_target_feature_enabled!(x)` acts like `cfg!(target_feature = x)`.

# Drawbacks
[drawbacks]: #drawbacks

- This RFC adds syntax sugar around the already relatively well-understood `#[target_feature]` attribute.
- Monomorphizing many instances of a function, while unlikely to occur, could be unexpected behavior.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Conditional compilation
[RFC #2045](https://github.com/rust-lang/rfcs/pull/2045) proprosed making `#[cfg(target_feature = "feature")]` context-dependent, but this was never implemented.
`cfg` is always consistent throughout a program, so making it context dependent might be confusing and lead to mistakes.
Additionally, it's not clear how context-dependent `#[cfg(target_feature = "feature")]` on items (rather than blocks) should or could work.
Adding a new `is_{arch}_feature_enabled` macro avoids this complexity.

## Multiversion
The [`multiversion` crate](https://crates.io/crates/multiversion) uses a macro to write multiple versions of functions for a list of target features.
A macro is limited to the scope of the annotated function, so it's not possible to select target features based on the caller.

## RFC #3449
This design is inspired by my previous attempt at solving the same problem, [RFC #3449: Contextual target feature detection](https://github.com/rust-lang/rfcs/pull/3449).

[Some insightful comments](https://github.com/rust-lang/rfcs/pull/3449#issuecomment-1596335701) raised concerns about the unreliability and complexity of relying on the inliner and backend to evaluate `is_{arch}_feature_enabled` so late in the compilation process.

This RFC resolves that concern by handling the process entirely within MIR.

## RFC #3528
[RFC #3528: Struct target features](https://github.com/rust-lang/rfcs/pull/3525) also proposes representing target features with generics, by using a struct annotated with the enabled target features.

Fundamentally, *struct target features* proposes using `#[target_feature]` without `unsafe`.
This has already been addressed by the approved and implemented [RFC #2396 (`#[target_feature]` 1.1")](https://github.com/rust-lang/rfcs/pull/2396).
This RFC, in comparison to *struct target features*, is compatible with TF1.1 and does not intend to replace it.

Using a struct tagged with `#[target_feature]` also has a few downsides:
- *Struct target features* requires boilerplate marker structs for each combination of target features. Each crate needs to define structs with the particular combination of features necessary.
- This RFC leverages the compiler to automatically insert the generic parameter in both the function signature and at the call site--*struct target features* requires adding it manually.
- `#[target_feature(caller)]` is explicit, readily visible, and universal compared to a struct with an arbitrary name.
- *Struct target features* needs to reject complex types to avoid soundness problems, e.g. `Option<Feature>` or `Vec<Feature>` cannot prove existence of a target feature.
- This RFC allows adjusting implementations based on the queried enabled target features. *Struct target features* requires specialization or cumbersome traits with associated consts.

# Prior art
[prior-art]: #prior-art

While not related to target features, `#[track_caller]` sets a precedent for passing caller information to a function through a parameter embedded by the compiler.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How should this interact with MIR inlining? Which target features should be used when the caller is inlined into another function?

# Future possibilities
[future-possibilities]: #future-possibilities

- As const generics are improved, the compiler-internal target features type could be exposed.
- `is_{arch}_feature_detected` could make use of this macro for additional optimization opportunity.
- Some form of "compile time `if`" could allow branches to safely call target feature functions using `target_feature_11`
