# Use-Case: Auditing Existing Code

`BikeshedIntrinsicFrom` can be used to audit the soundness of existing transmutations in code-bases. This macro demonstrates a drop-in replacement to `mem::transmute` that produces a compile error if the transmutation is unsound:
```rust,ignored
macro_rules! transmute {
    ($src:expr) => {transmute!($src, Assume {})};
    ($src:expr, Assume { $( $assume:ident ),* } ) => {{
        #[inline(always)]
        unsafe fn transmute<Src, Dst, Scope, const ASSUME: Assume>(src: Src) -> Dst
        where
            Dst: BikeshedIntrinsicFrom<Src, Scope, ASSUME>
        {
            #[repr(C)]
            union Transmute<Src, Dst> {
                src: ManuallyDrop<Src>,
                dst: ManuallyDrop<Dst>,
            }

            ManuallyDrop::into_inner(Transmute { src: ManuallyDrop::new(src) }.dst)
        }

        struct Scope;

        const ASSUME: Assume = {
            let mut assume = Assume::NOTHING;
            $(assume . $assume = true;)*
            assume
        };

        transmute::<_, _, Scope, ASSUME>($src)
    }};
}
```

For example, consider this use of `mem::transmute`:
```rust,ignore
unsafe fn foo(v: u8) -> bool {
    mem::transmute(v)
}
```
Swapping `mem::transmute` out for our macro (rightfully) produces a compile error:
```rust,ignore
unsafe fn foo(v: u8) -> bool {
    unsafe { transmute!(v) } // Compile Error!
}
```
...that we may resolve by explicitly instructing the compiler to assume validity:
```rust,ignore
fn foo(v: u8) -> bool {
    assert!(v < 2);
    unsafe { transmute!(v, Assume { validity }) }
}
```

