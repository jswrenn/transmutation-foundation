# How should `Assume` values be created, combined?

To initialize and combine `Assume` values, this proposal defines a set of associated constants and an `Add` impl:
```rust,ignore
impl Assume {
    pub const NOTHING: Self = Self {
        alignment   : false,
        lifetimes   : false,
        validity    : false,
        visibility  : false,
    };

    pub const ALIGNMENT:  Self = Self {alignment: true, ..Self::NOTHING};
    pub const LIFETIMES:  Self = Self {lifetimes: true, ..Self::NOTHING};
    pub const VALIDITY:   Self = Self {validity:  true, ..Self::NOTHING};
    pub const VISIBILITY: Self = Self {validity:  true, ..Self::NOTHING};
}

impl const core::ops::Add for Assume {
    type Output = Self;

    fn add(self, other: Self) -> Self {
        Self {
            alignment   : self.alignment  || other.alignment,
            lifetimes   : self.lifetimes  || other.lifetimes,
            validity    : self.validity   || other.validity,
            visibility  : self.visibility || other.visibility,
        }
    }
}
```

Consequently, `Assume` values can be ergonomically created (e.g., `Assume::ALIGNMENT`) and combined (e.g., `Assume::ALIGNMENT + Assume::VALIDITY + ASSUME`).

Let's contrast this approach with two other possibilities:

## Alternative: Minimalism
Alternatively, we might only provide:
```rust,ignore
impl Assume {
    pub const NOTHING: Self = Self {
        alignment   : false,
        lifetimes   : false,
        validity    : false,
        visibility  : false,
    };
}
```
This is the minimum impl we must provide for `Assume` to be useful. With it, `Assume` values can be created:
```rust
const ASSUME_ALIGNMENT: Assume = {
  let mut assume = Assume::NOTHING;
  assume.alignment = true;
  assume
};
```
and combined:
```rust
const ALSO_ASSUME_ALIGNMENT_VALIDITY: Assume = {
  let mut assume = ASSUME;
  assume.alignment = true;
  assume.validity = true;
  assume
};
```

This approach achieves minimalism at the cost of ergonomics.

## Alternative: Builder Methods
Alternatively, we could define chainable builder methods:
```rust,ignore
impl Assume {
    pub const NOTHING: Self = Assume {
        alignment   : false,
        lifetimes   : false,
        validity    : false,
        visibility  : false,
    };

    pub const fn alignment(self)  -> Self { Assume { alignment:  true, ..self } }
    pub const fn lifetimes(self)  -> Self { Assume { lifetimes:  true, ..self } }
    pub const fn validity(self)   -> Self { Assume { validity:   true, ..self } }
    pub const fn visibility(self) -> Self { Assume { visibility: true, ..self } }
}
```
With this, `Assume` values can be created:
```rust,ignore
Assume::NOTHING.alignment()
```
...and combined:
```rust,ignore
ASSUME.alignment().validity()
```

This approach is almost as succinct as the approach selected by this proposal (i.e., the `Add` impl), but meaning of the resulting expressions are not quite as self-evident.
