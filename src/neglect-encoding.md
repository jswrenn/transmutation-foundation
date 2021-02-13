# Why is Neglect a const parameter?
We've considered [a few different ways](https://hackmd.io/@jswrenn/S192QCR9D) in which `Neglect` might be represented. An ideal representation has three properties:
 1. is defaultable with "neglect nothing", so doing the totally safe thing is truly the easiest thing
 2. every subset of neglected options has exactly *one* encoding (i.e., neglecting validity and alignment is the same as neglecting alignment and validity)
 3. is generically adjustable (i.e., I can add or remove a neglect from an existing set of options)

The type parameter approaches we've considered tick both the first box, and *either* the second xor third. The sorts of type system extensions that would permit ticking all three of these boxes with a type parameter are far off.

In contrast:
1. A const generic `Neglect` parameter is *not* (yet) defaultable, but this doesn't seem like a permanent limitation of const generics.
2. The const generic `Neglect` admits exactly one encoding of each subset. The value produced by `Neglect {alignment: true, validity: true}` is identical to the value produced by `Neglect {validity: true, alignment: true}`.
3. The const generic `Neglect` is generically extendable. Given an existing, arbitrary `NEGLECT`, we additionally can disable the alignment check with `{Neglect::ALIGNMENT + NEGLECT}`.
