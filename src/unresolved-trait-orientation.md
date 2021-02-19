# Trait Orientation: `From` or `Into`?

Should it be `BikeshedIntrinsicFrom` or `BikeshedIntrinsicInto`? What factors should be considered?


## `From`
```rust,ignore
pub unsafe trait BikeshedIntrinsicFrom<Src, Scope, const ASSUME: Assume>
where
    Src: ?Sized
{
    unsafe fn unsafe_bikeshed_from(src: Src) -> Self
    where
        Src: Sized,
        Self: Sized,
    {
        #[repr(C)]
        union Transmute<Src, Dst> {
            src: ManuallyDrop<Src>,
            dst: ManuallyDrop<Dst>,
        }

        ManuallyDrop::into_inner(Transmute { src: ManuallyDrop::new(src) }.dst)
    }
}
```

## `Into`
```rust,ignore
pub unsafe trait BikeshedIntrinsicInto<Dst, Scope, const ASSUME: Assume>
where
    Dst: ?Sized
{
    unsafe fn unsafe_bikeshed_into(self) -> Dst
    where
        Self: Sized,
        Dst: Sized,
    {
        #[repr(C)]
        union Transmute<Src, Dst> {
            src: ManuallyDrop<Src>,
            dst: ManuallyDrop<Dst>,
        }

        ManuallyDrop::into_inner(Transmute { src: ManuallyDrop::new(self) }.dst)
    }
}
```