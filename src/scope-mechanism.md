# How does `Scope` ensure safety?

It's generally unsound to construct instances of types for which you do not have a constructor.

If `BikeshedIntrinsicFrom` *lacked* a `Scope` parameter; e.g.,:
```rust,ignore
// we'll also omit `ASSUME` for brevity
pub unsafe trait BikeshedIntrinsicFrom<Src>
where
    Src: ?Sized
{}
```
...we could not use it to check the soundness of the transmutations in this example:
```rust,ignore
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

## Solution

We resolve this inconsistency by introducing a type parameter, `Scope`, that allows Rust to distinguish between these two contexts:
```rust,ignore
// we omit `ASSUME` for brevity
pub unsafe trait BikeshedIntrinsicFrom<Src, Scope>
where
    Src: ?Sized
{}
```
`Scope` must be instantiated with any private (i.e., `pub(self)` type. To determine whether a transmutation is safe, the compiler pretends that it is at the defining scope of that type, and checks that the necessary fields of `Src` and `Dst` are visible.

For example:
```rust,ignore
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

