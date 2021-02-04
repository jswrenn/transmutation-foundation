# Why do we need Neglect?
The ability to neglect particular static checks makes `BikeshedIntrinsicFrom` useful in scenarios where aspects of well-definedness and safety are ensured through other means.

## Example: Neglecting Alignment
For the instantiation of a `&'dst Dst` from any `&'src Src` to be safe, the minimum required alignment of all `Src` must be stricter than the minimum required alignment of `Dst`, among other factors (e.g., `'src` must outlive `'dst`). By default, `BikeshedIntrinsicFrom` will enforce these requirements statically.

However, for the instantiation of a `&'dst Dst` from a *particular* `&'src Src` to be safe, we can just check that the alignment of that *particular* `&'src Src` is sufficient using `mem::align_of` (e.g., see bytemuck's [try_cast_ref](https://docs.rs/bytemuck/1.4.1/bytemuck/fn.try_cast_ref.html) method).

Using a `NEGLECT` parameter of `Neglect {alignment: true, ..Neglect::NOTHING}` makes `BikeshedIntrinsicFrom` useful in this scenario. With that `NEGLECT` parameter, `BikeshedIntrinsicFrom` neglects only its static alignment check, which we then assume responsibility to enforce ourselves; e.g.:
```rust,ignore

const NEGLECT_ALIGNMENT : NEGLECT = Neglect {alignment: true, ..Neglect::NOTHING};

/// Try to convert a `&'src Src` into `&'dst Dst`.
///
/// This produces `None` if the referent isn't appropriately
/// aligned, as required by the destination type.
fn try_cast_ref<'src, 'dst, Src, Dst, Scope>(src: &'src Src) -> Option<&'dst Dst>
where
    &'t T: BikeshedIntrinsicFrom<&'dst Dst, Scope, NEGLECT_ALIGNMENT>,
{
    // check alignment dynamically
    if (src as *const Src as usize) % align_of::<Dst>() != 0 {
        None
    } else {
        // SAFETY: we've dynamically enforced the alignment requirement.
        // `BikeshedIntrinsicFrom` statically enforces all other safety reqs.
        unsafe { &*(src as *const Src as *const Dst) }
    }
}
```