# Visibility of Src?

In the below example, is `downstream::as_bytes` sound?
```rust,ignored
mod upstream {
    /// Implement this to promise that your type's layout consists
    /// solely of initialized bytes with no library invariants.
    pub unsafe trait POD {}
}

mod downstream {
    use super::*;

    pub fn as_bytes<'t, T>(t: &'t T) -> &'t [T]
    where
        T: upstream::POD,
    {
        use core::{slice, mem::size_of};
        
        unsafe {
            slice::from_raw_parts(t as *const T, size_of::<T>())
        }
    }
}
```

The answer to this question impacts the implementation details of `BikeshedIntrinsicFrom`.

**If yes**, then the [three aforementioned visibility conditions](introduction.md#when-is-a-transmutation-well-defined-and-safe) are sufficient.

**If no**, there is a fourth condition necessary for safety:
> In order to be **safe**, a well-defined transmutation must also not allow you to:
> 1. construct instances of a hidden `Dst` type
> 2. mutate hidden fields of the `Src` type
> 3. construct hidden fields of the `Dst` type
> 4. ***read hidden fields of the `Src` type***



