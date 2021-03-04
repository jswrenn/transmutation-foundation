# Proposal

Add a compiler-intrinsic trait be added to `core::mem` for checking the soundness of bit-reinterpretation casts (e.g., `mem::transmute`, `union`, pointer casting, etc.):
```rust
pub unsafe trait BikeshedIntrinsicFrom<Src, Context, const ASSUME: Assume>
where
    Src: ?Sized
{}

#[derive(PartialEq, Eq)]
#[non_exhaustive]
pub struct Assume {
    pub alignment   : bool,
    pub lifetimes   : bool,
    pub validity    : bool,
    pub visibility  : bool,
}

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

    fn add(self, rhs: Self) -> Self {
        Self {
            alignment   : self.alignment  || rhs.alignment,
            lifetimes   : self.lifetimes  || rhs.lifetimes,
            validity    : self.validity   || rhs.validity,
            visibility  : self.visibility || rhs.visibility,
        }
    }
}

impl const core::ops::Sub for Assume {
    type Output = Self;

    fn add(self, rhs: Self) -> Self {
        Self {
            alignment   : self.alignment  && !rhs.alignment,
            lifetimes   : self.lifetimes  && !rhs.lifetimes,
            validity    : self.validity   && !rhs.validity,
            visibility  : self.visibility && !rhs.visibility,
        }
    }
}
```

## When is a bit-reinterpretation cast sound?

A bit-reinterpretation cast (henceforth: "transmutation") is sound if it is both well-defined and safe.

A transmutation is **well-defined** if *any* possible values of type `Src` are a valid instance of `Dst`. The compiler determines this by inspecting the layouts of `Src` and `Dst`.

In order to be **safe**, any safe use of the transmutation result cannot cause memory unsafety. Namely, a transmutation is generally unsafe if it allows you to:
1. construct instances of a hidden `Dst` type
2. mutate hidden fields of the `Src` type
3. construct hidden fields of the `Dst` type

Whether these conditions are satisfied depends on the scope the transmutation occurs in. The existing mechanism of [type privacy](https://rust-lang.github.io/rfcs/2145-type-privacy.html) will ensure that first condition is satisfied. To enforce the second and third conditions, we introduce the `Context` type parameter (see below). 

## What is `Assume`?

The `Assume` parameter encodes the set of properties that the compiler should assume (rather than check) when determining transmutability. These checks include:
- alignment
- lifetimes
- validity
- visibility

The ability to omit particular static checks makes `BikeshedIntrinsicFrom` useful in scenarios where aspects of well-definedness and safety are ensured through other means (e.g., domain knowledge or runtime checks).

## What is `Context`?

The `Context` parameter of `BikeshedIntrinsicFrom` is used to ensure that the second and third safety conditions are satisfied.

When visibility is enforced, `Context` must be instantiated with any private (i.e., `pub(self)` type. The compiler pretends that it is at the defining scope of that type, and checks that the necessary fields of `Src` and `Dst` are visible.

When visibility is assumed, the `Context` parameter is ignored.

### Why is safety dependent on context?

In order to be safe, a well-defined transmutation must also not allow you to:
1. construct instances of a hidden `Dst` type
2. mutate hidden fields of the `Src` type
3. construct hidden fields of the `Dst` type

Whether these conditions are satisfied depends on the context of the transmutation, because scope determines the visibility of fields. Consider:
```rust
mod a {
    mod npc {
        #[repr(C)]
        pub struct NoPublicConstructor(u32);
        
        impl NoPublicConstructor {
            pub(super) fn new(v: u32) -> Self {
                assert!(v % 2 == 0);
                unsafe { core::mem::transmute(v) } // okay.
            }

            pub fn method(self) {
                if self.0 % 2 == 1 {
                    // totally unreachable, thanks to assert in `Self::new`
                    unsafe { *std::ptr::null() }
                }
            }
        }
    }

    use npc::NoPublicConstructor;
}

mod b {
    use super::*;

    fn new(v: u32) -> a::NoPublicConstructor {
        unsafe { core::mem::transmute(v) } // ☢️ BAD!
    }
}
```
The function `b::new` is unsound, because it constructs an instance of a type without a public constructor.

### How does `Context` ensure safety?

It's generally unsound to construct instances of types for which you do not have a constructor. If `BikeshedIntrinsicFrom` *lacked* a `Context` parameter; e.g.,:
```rust
// we'll also omit `ASSUME` for brevity
pub unsafe trait BikeshedIntrinsicFrom<Src>
where
    Src: ?Sized
{}
```
...we could not use it to check the soundness of the transmutations in this example:
```rust
mod a {
    use super::*;

    mod npc {
        #[repr(C)]
        pub struct NoPublicConstructor(u32);
        
        impl NoPublicConstructor {
            pub(super) fn new(v: u32) -> Self {
                assert!(v % 2 == 0);
                assert_impl!(NoPublicConstructor: BikeshedIntrinsicFrom<u32>);
                unsafe { core::mem::transmute(v) } // okay.
            }

            pub fn method(self) {
                if self.0 % 2 == 1 {
                    // totally unreachable, thanks to assert in `Self::new`
                    unsafe { *std::ptr::null() }
                }
            }
        }
    }

    use npc::NoPublicConstructor;
}

mod b {
    use super::*;

    fn new(v: u32) -> a::NoPublicConstructor {
        assert_not_impl!(NoPublicConstructor: BikeshedIntrinsicFrom<u32>);
        unsafe { core::mem::transmute(v) } // ☢️ BAD!
    }
}
```
In module `a`, `NoPublicConstructor` must implement `BikeshedIntrinsicFrom<u32>`. In module `b`, it must not. This inconsistency is incompatible with Rust's trait system.

#### Solution

We resolve this inconsistency by introducing a type parameter, `Context`, that allows Rust to distinguish between these two contexts:
```rust
// we omit `ASSUME` for brevity
pub unsafe trait BikeshedIntrinsicFrom<Src, Context>
where
    Src: ?Sized
{}
```
`Context` must be instantiated with any private (i.e., `pub(self)` type. To determine whether a transmutation is safe, the compiler pretends that it is at the defining scope of that type, and checks that the necessary fields of `Src` and `Dst` are visible.

For example:
```rust
mod a {
    use super::*;

    mod npc {
        #[repr(C)]
        pub struct NoPublicConstructor(u32);
        
        impl NoPublicConstructor {
            pub(super) fn new(v: u32) -> Self {
                assert!(v % 2 == 0);
                struct A; // a private type that represents this context
                assert_impl!(NoPublicConstructor: BikeshedIntrinsicFrom<u32, A>);
                unsafe { core::mem::transmute(v) } // okay.
            }

            pub fn method(self) {
                if self.0 % 2 == 1 {
                    // totally unreachable, thanks to assert in `Self::new`
                    unsafe { *std::ptr::null() }
                }
            }
        }
    }

    use npc::NoPublicConstructor;
}

mod b {
    use super::*;

    fn new(v: u32) -> a::NoPublicConstructor {
        struct B; // a private type that represents this context
        assert_not_impl!(NoPublicConstructor: BikeshedIntrinsicFrom<u32, B>);
        unsafe { core::mem::transmute(v) } // ☢️ BAD!
    }
}
```

In module `a`, `NoPublicConstructor` implements `BikeshedIntrinsicFrom<u32, A>`. In module `b`, `NoPublicConstructor` does *not* implement `BikeshedIntrinsicFrom<u32, B>`. There is no inconsistency.

### Can't Context be elided?

*Not generally.* Consider a hypothetical `FromZeros` trait that indicates whether `Self` is safely initializable from a sufficiently large buffer of zero-initialized bytes:
```rust
pub mod zerocopy {
    pub unsafe trait FromZeros<const ASSUME: Assume> {
        /// Safely initialize `Self` from zeroed bytes.
        fn zeroed() -> Self;
    }

    #[repr(u8)]
    enum Zero {
        Zero = 0u8
    }

    unsafe impl<Dst, const ASSUME: Assume> FromZeros<ASSUME> for Dst
    where
        Dst: BikeshedIntrinsicFrom<[Zero; mem::MAX_OBJ_SIZE], ???, ASSUME>,
    {
        fn zeroed() -> Self {
            unsafe { mem::transmute([Zero; size_of::<Self>]) }
        }
    }
}
```
The above definition leaves ambiguous (`???`) the context in which the constructability of `Dst` is checked: is it from the perspective of where this trait is defined, or where this trait is *used*? In this example, you probably do *not* intend for this trait to *only* be usable with `Dst` types that are defined in the same scope as the `FromZeros` trait!

An explicit `Context` parameter on `FromZeros` makes this unambiguous; the transmutability of `Dst` should be assessed from where the trait is used, *not* where it is defined:
```rust
pub unsafe trait FromZeros<Context, const ASSUME: Assume> {
    /// Safely initialize `Self` from zeroed bytes.
    fn zeroed() -> Self;
}

unsafe impl<Dst, Context, const ASSUME: Assume> FromZeros<Context, ASSUME> for Dst
where
    Dst: BikeshedIntrinsicFrom<[Zero; usize::MAX], Context, ASSUME>
{
    fn zeroed() -> Self {
        unsafe { mem::transmute([Zero; size_of::<Self>]) }
    }
}
```

## External Documents and Discussion

This proposal:
- [supplementary design documentation](https://jswrenn.github.io/transmutation-foundation/)
- [discussion in #t-lang "transmutability"](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/transmutability)

Prior proposal:
- [RFC2981](https://github.com/rust-lang/rfcs/pull/2981) (superceeded by this MCP)

# Mentors or Reviewers

*If you have a reviewer or mentor in mind for this work, mention then
here. You can put your own name here if you are planning to mentor the
work.*

# Process

The main points of the [Major Change Process][MCP] is as follows:

* [x] File an issue describing the proposal.
* [ ] A compiler team member or contributor who is knowledgeable in the area can **second** by writing `@rustbot second`.
    * Finding a "second" suffices for internal changes. If however you are proposing a new public-facing feature, such as a `-C flag`, then full team check-off is required.
    * Compiler team members can initiate a check-off via `@rfcbot fcp merge` on either the MCP or the PR.
* [ ] Once an MCP is seconded, the Final Comment Period begins. If no objections are raised after 10 days, the MCP is considered **approved**.

You can read [more about Major Change Proposals on forge][MCP].

# Comments

**This issue is not meant to be used for technical discussion. There is a Zulip stream for that. Use this issue to leave procedural comments, such as volunteering to review, indicating that you second the proposal (or third, etc), or raising a concern that you would like to be addressed.**

[MCP]: https://forge.rust-lang.org/compiler/mcp.html
[forge]: https://forge.rust-lang.org/
