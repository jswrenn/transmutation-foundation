# Overview

We propose that this *compiler-intrinsic* trait (end-users *cannot* implement it) be added to `core::mem`:
```rust,ignore
pub unsafe trait BikeshedIntrinsicFrom<Src, Scope, const ASSUME: Assume>
where
    Src: ?Sized
{}
```

This trait is capable of telling you whether a particular bit-reinterpretation cast from `Src` to `Self` is well-defined and safe (notwithstanding whatever static checks you decide to `Assume`).

This trait is useful:
1. [for auditing the safety of existing code](./use-case-auditing.md)
2. [as a foundation for high-level abstractions](./use-case-abstraction.md)


## When is a transmutation well-defined and safe?
A transmutation is **well-defined** if *any* possible values of type `Src` are a valid instance of `Dst`. The compiler determines this by inspecting the layouts of `Src` and `Dst`.

In order to be **safe**, a well-defined transmutation must also not allow you to:
1. construct instances of a hidden `Dst` type
2. mutate hidden fields of the `Src` type
3. construct hidden fields of the `Dst` type

Whether these conditions are satisfied depends on the scope the transmutation occurs in. The existing mechanism of [type privacy](https://rust-lang.github.io/rfcs/2145-type-privacy.html) will ensure that first condition is satisfied. To enforce the second and third conditions, we introduce the `Scope` type parameter (see below). 

## What is `Assume`?
The `Assume` parameter encodes the set of static checks that the compiler should ignore when determining transmutability. These checks include:
- alignment
- lifetimes
- validity
- visibility

The `Assume` type is represented like this:
```rust,ignore
#[derive(PartialEq, Eq)]
#[non_exhaustive]
pub struct Assume {
    pub alignment   : bool,
    pub lifetimes   : bool,
    pub validity    : bool,
    pub visibility  : bool,
}

impl Assume {
    pub const NOTHING: Self = Self {
        alignment   : false,
        lifetimes   : false,
        validity    : false,
        visibility  : false,
    };

    pub const ALIGNMENT:  Self = Self {alignment: true, ..Self::NOTHING};
    pub const LIFETIMES:  Self = Self {lifetimes: true, ..Self::NOTHING};
    pub const VALIDITY:   Self = Self {validity:  true, ..Self::NOTHING};
    pub const VISIBILITY: Self = Self {validity:  true, ..Self::NOTHING};
}

impl const core::ops::Add for Assume {
    type Output = Self;

    fn add(self, other: Self) -> Self {
        Self {
            alignment   : self.alignment  || other.alignment,
            lifetimes   : self.lifetimes  || other.lifetimes,
            validity    : self.validity   || other.validity,
            visibility  : self.visibility || other.visibility,
        }
    }
}
```

**For more information, see [here](options.md).**

## What is `Scope`?
The `Scope` parameter of `BikeshedIntrinsicFrom` is used to ensure that the second and third safety conditions are satisfied.

When visibility is enforced, `Scope` must be instantiated with any private (i.e., `pub(self)` type. The compiler pretends that it is at the defining scope of that type, and checks that the necessary fields of `Src` and `Dst` are visible.

When visibility is assumed, the `Scope` parameter is ignored.

**For more information, see [here](scope.md).**