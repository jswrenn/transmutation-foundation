# How does `Scope` ensure safety?

If `BikeshedIntrinsicFrom` *lacked* a `Scope` parameter; e.g.,:
```rust,ignore
// we'll also omit `NEGLECT` for brevity
pub unsafe trait BikeshedIntrinsicFrom<Src>
where
    Src: ?Sized
{}
```
...we could not use it to check the soundness of the transmutations in this example:
```rust,ignore
mod a {
    use super::*;

    #[repr(C)]
    pub struct NonZeroU32(u32);
    
    impl NonZeroU32 {
        fn new(v: u32) -> Self {
            assert_impl!(NonZeroU32: BikeshedIntrinsicFrom<u32>);
            unsafe { core::mem::transmute(v) } // sound.
        }
    }
}

mod b {
    use super::*;

    fn new(v: u32) -> a::NonZeroU32 {
        assert_not_impl!(NonZeroU32: BikeshedIntrinsicFrom<u32>);
        unsafe { core::mem::transmute(v) } // ☢️ UNSOUND!
    }
}
```
In module `a`, `NonZeroU32` must implement `BikeshedIntrinsicFrom<u32>`. In module `b`, it must not. This inconsistency is incompatible with Rust's trait system.

## Solution

We resolve this inconsistency by introducing a type parameter, `Scope`, that allows Rust to distinguish between these two contexts:
```rust,ignore
// we omit `NEGLECT` for brevity
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

    #[repr(C)]
    pub struct NonZeroU32(u32);
    
    impl NonZeroU32 {
        fn new(v: u32) -> Self {
            struct A; // a private type that represents this context
            assert_impl!(NonZeroU32: BikeshedIntrinsicFrom<u32, A>);
            unsafe { core::mem::transmute(v) } // sound.
        }
    }
}

mod b {
    use super::*;

    fn new(v: u32) -> a::NonZeroU32 {
        struct B; // a private type that represents this context
        assert_not_impl!(NonZeroU32: BikeshedIntrinsicFrom<u32, B>);
        unsafe { core::mem::transmute(v) } // ☢️ UNSOUND!
    }
}
```

In module `a`, `NonZeroU32` implements `BikeshedIntrinsicFrom<u32, A>`. In module `b`, `NonZeroU32` does *not* implement `BikeshedIntrinsicFrom<u32, B>`. There is no inconsistency.

