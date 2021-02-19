# Use-Case: Abstraction

Like `size_of` and `align_of`, `BikeshedIntrinsicFrom` is not SemVer-conscious. However, we can use it as the foundation for a variety of SemVer-conscious APIs.

## Example: Muckable
In this example, end-users implement the unsafe marker trait `Muckable` to denote can be cast (via `MuckFrom`) from or into any other compatible, `Muckable` type:
```rust,ignore
/// Implemented by user to denote that the type and its fields (recursively):
///   - promise complete layout stability
///   - have no library invariants on their values
pub unsafe trait Muckable {}

/// Implemented if `Self` can be mucked from the bits of `Src`.
pub unsafe trait MuckFrom<Src>
{
    fn muck_from(src: Src) -> Self
    where
        Self: Sized;
}

unsafe impl<Src, Dst> MuckFrom<Src> for Dst
where
    Src: Muckable,
    Dst: Muckable,
    Dst: BikeshedIntrinsicFrom<Src, !, {Assume::VISIBILITY}>
{
    fn muck_from(src: Src) -> Self
    where
        Self: Sized,
    {
        #[repr(C)]
        union Transmute<Src, Dst> {
            src: ManuallyDrop<Src>,
            dst: ManuallyDrop<Dst>,
        }

        unsafe { ManuallyDrop::into_inner(Transmute { src: ManuallyDrop::new(src) }.dst) }
    }
}
```
