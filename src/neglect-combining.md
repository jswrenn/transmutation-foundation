# How should `Neglect` values be created, combined?

To initialize and combine `Neglect` values, this proposal defines a set of associated constants and an `Add` impl:
```rust,ignore
impl Neglect {
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

impl const core::ops::Add for Neglect {
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

Consequently, `Neglect` values can be ergonomically created (e.g., `Neglect::ALIGNMENT`) and combined (e.g., `Neglect::ALIGNMENT + Neglect::VALIDITY + NEGLECT`).

Let's contrast this approach with two other possibilities:

## Alternative: Minimalism
Alternatively, we might only provide:
```rust,ignore
impl Neglect {
    pub const NOTHING: Self = Self {
        alignment   : false,
        lifetimes   : false,
        validity    : false,
        visibility  : false,
    };
}
```
This is the minimum impl we must provide for `Neglect` to be useful. With it, `Neglect` values can be created:
```rust
const NEGLECT_ALIGNMENT: Neglect = {
  let mut neglect = Neglect::NOTHING;
  neglect.alignment = true;
  neglect
};
```
and combined:
```rust
const ALSO_NEGLECT_ALIGNMENT_VALIDITY: Neglect = {
  let mut neglect = NEGLECT;
  neglect.alignment = true;
  neglect.validity = true;
  neglect
};
```

This approach achieves minimalism at the cost of ergonomics.

## Alternative: Builder Methods
Alternatively, we could define chainable builder methods:
```rust,ignore
impl Neglect {
    pub const NOTHING: Self = Neglect {
        alignment   : false,
        lifetimes   : false,
        validity    : false,
        visibility  : false,
    };

    pub const fn alignment(self)  -> Self { Neglect { alignment:  true, ..self } }
    pub const fn lifetimes(self)  -> Self { Neglect { lifetimes:  true, ..self } }
    pub const fn validity(self)   -> Self { Neglect { validity:   true, ..self } }
    pub const fn visibility(self) -> Self { Neglect { visibility: true, ..self } }
}
```
With this, `Neglect` values can be created:
```rust,ignore
Neglect::NOTHING.alignment()
```
...and combined:
```rust,ignore
NEGLECT.alignment().validity()
```

This approach is almost as succinct as the approach selected by this proposal (i.e., the `Add` impl), but meaning of the resulting expressions are not quite as self-evident.
