# Can't Context be elided?

**Not generally.**

Consider a hypothetical `FromZeros` trait that indicates whether `Self` is safely initializable from a sufficiently large buffer of zero-initialized bytes:
```rust,ignore
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
```rust,ignore
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