# Changelog

All notable changes to units-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `undim` — a dimension as seven exponents, the algebra over them, and
  the `n`-th root as the one operation in it that can fail.
- `unqty` — twenty-three `@value` quantity types carrying one Float in
  the SI base unit, the arithmetic between dimensions as named
  functions, and not a single `Result`.
- `undyn` — a quantity whose dimension is a value, the arithmetic that
  checks it at run time, and the bridge that answers a raw magnitude.
- `unconv` — units as a scale, an offset and a dimension, the tables
  per dimension, and the two temperature conversions.
- `untext` — the grammar whole, parse-then-format as the identity, and
  formatting that is not localised on purpose.
- `unfault` — every refusal, all of them in the dynamic half.

### Known

- **`UnDim` is the load-bearing interface** and it is boxed, which it
  should not have to be: a `@value` struct may not be a field of a
  boxed struct nor an enum payload.
- **The static/dynamic line is drawn by the representation**, not by
  taste: `@value` survives at the leaves, where a quantity is one Float
  and is never carried, compared with an operator, or returned
  fallibly.
- **novo-lang has no arithmetic operator overloading**, so every
  operation is a named function — including equality, which a `@value`
  type cannot spell with `==` at all.
- **`UnTemperature` and `UnTempDelta` are two types.** The offset
  applies to one and not to the other, and an offset unit in a compound
  expression is refused rather than resolved.
- **The static set is a set, not a parameter**: there are no const
  generics and no numeric bound a `@value` struct could carry.
- **No `@tier(embedded)` claim**, though the static half would fit one;
  a `units-core-nv` split is named in the README.
- **No dependencies.**
