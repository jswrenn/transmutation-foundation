# Why is Assume a const parameter?
We've considered [a few different ways](https://hackmd.io/@jswrenn/S192QCR9D) in which `Assume` might be represented. An ideal representation has three properties:
 1. is defaultable with "assume nothing", so doing the totally safe thing is truly the easiest thing
 2. every subset of assumptions has exactly *one* encoding (i.e., assuming validity and alignment is the same as assuming alignment and validity)
 3. is generically adjustable (i.e., I can add or remove an assumption from an existing set of options)

The type parameter approaches we've considered tick both the first box, and *either* the second xor third. The sorts of type system extensions that would permit ticking all three of these boxes with a type parameter are far off.

In contrast:
1. A const generic `Assume` parameter is *not* (yet) defaultable, but this doesn't seem like a permanent limitation of const generics.
2. The const generic `Assume` admits exactly one encoding of each subset. The value produced by `Assume {alignment: true, validity: true}` is identical to the value produced by `Assume {validity: true, alignment: true}`.
3. The const generic `Assume` is generically extendable. Given an existing, arbitrary `ASSUME`, we additionally can disable the alignment check with `{Assume::ALIGNMENT + ASSUME}`.
