# Use-Case: Auditing Existing Code

`BikeshedIntrinsicFrom` can be used to audit the soundness of existing transmutations in code-bases. This macro demonstrates a drop-in replacement to `mem::transmute` that produces a compile error if the transmutation is unsound:
```rust,ignored
macro_rules! transmute {
    ($src:expr) => {transmute!($src, Neglect {})};
    ($src:expr, Neglect { $( $neglect:ident ),* } ) => {{
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

        struct Scope;

        const NEGLECT: Neglect = Neglect { $($neglect: true,)* ..Neglect::NOTHING};

        transmute::<_, _, Scope, NEGLECT>($src)
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
    transmute!(v) // Compile Error!
}
```
...that we may resolve by explicitly neglecting validity:
```rust,ignore
fn foo(v: u8) -> bool {
    assert!(v < 2);
    transmute!(v, Neglect { validity })
}
```

