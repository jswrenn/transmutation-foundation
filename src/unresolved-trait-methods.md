# Should the trait have methods?

Many use-cases of `BikeshedIntrinsicFrom` involve using something like this `BikeshedIntrinsicFrom`-bounded function:
```rust,ignore
#[inline(always)]
unsafe fn transmute<Src, Dst, Scope, const NEGLECT: Neglect>(src: Src) -> Dst
where
    Dst: BikeshedIntrinsicFrom<Src, Scope, NEGLECT>
{

    #[repr(C)]
    union Transmute<Src, Dst> {
        src: ManuallyDrop<Src>,
        dst: ManuallyDrop<Dst>,
    }

    ManuallyDrop::into_inner(Transmute { src: ManuallyDrop::new(src) }.dst)
}
```
Defining this function *could* be left as an exercise to the end-user. Or, `mem` could provide it. **We don't need to resolve this for the initial proposal**, but having an inkling of how we'd like to tackle it may affect how we name and structure the items defined by this proposal.

That function, as defined, cannot be added to the root of `mem`, because it would conflict with `mem::transmute`.

We could resolve this conflict by:
1. using a [name](./unresolved-trait-name.md) other than "transmute" for this proposal.
2. placing `BikeshedIntrinsicFrom`, `Neglect`, and this function under a new module (e.g., `mem::cast`)
3. defining an associated function on `BikeshedIntrinsicFrom`; e.g.:   
    ```rust,ignore
    pub unsafe trait BikeshedIntrinsicFrom<Src, Scope, const NEGLECT: Neglect>
    where
        Src: ?Sized
    {
        unsafe fn unsafe_bikeshed_from(src: Src) -> Self
        where
            Src: Sized
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
    **Selecting this option might have ergonomic implications for the [orientation](unresolved-trait-orientation.md) of our trait.**
