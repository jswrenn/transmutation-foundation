# What should the name of `mem::BikeshedIntrinsicFrom` be?

## Choice: Verb vs. Adjective
E.g., `TransmuteFrom` or `TransmutableFrom`?

Most trait names can be read as verbs (e.g., `convert::From`, `Send`, `Sync`), but `Sized` is a notable exception. Which convention should our trait follow?

## Choice: Root Word
E.g., `TransmuteFrom` or `BitsFrom`?

We have many options for root word:
- `Transmute`
- `Reinterpret`
- `Cast`
- `Bits`
- `Bytes`

## Choice: Adjective Prefix?
E.g., `TransmuteFrom` or `IntrinsicTransmuteFrom`?

Should the root word be prefixed with another word?

If so, what? Some possibilities:
- `Intrinsic`
- `Raw`
- `Is`
