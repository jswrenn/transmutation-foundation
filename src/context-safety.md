# Why is safety dependent on context?
In order to be safe, a well-defined transmutation must also not allow you to:
1. construct instances of a hidden `Dst` type
2. mutate hidden fields of the `Src` type
3. construct hidden fields of the `Dst` type

Whether these conditions are satisfied depends on the context of the transmutation, because scope determines the visibility of fields. Consider:
```rust
mod a {
    use super::*;

    #[repr(C)]
    pub struct NonZeroU32(u32);

    impl NonZeroU32 {
        fn new(v: u32) -> Self {
            unsafe { core::mem::transmute(v) } // sound.
        }
    }
}

mod b {
    use super::*;

    fn new(v: u32) -> a::NonZeroU32 {
        unsafe { core::mem::transmute(v) } // ☢️ UNSOUND!
    }
}
```

The transmutation in `b` is unsound because it constructs a hidden field of `NonZeroU32`.
