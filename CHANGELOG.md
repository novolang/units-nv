# Changelog

All notable changes to units-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-11

The bodies land: every function under `src/` is implemented,
`stability = "experimental"`, and the `novo` floor is the 0.20.2 the
release was verified against.  The interface is unchanged — not one
signature, effect row or published name moved.

### Added

- `undim` — the dimension algebra over the seven exponents, with
  `dim_root` as the one operation that fails, and the two renderings
  (`m·s⁻²` for a person, `m/s^2` for a machine).
- `unqty` — the ninety bodies of the typed half: the twenty-three
  constructors and their accessors, the eighteen physical relations,
  the arithmetic within a type, the named comparisons, and the four
  temperature operations.  Not one `Result`.
- `undyn` — the dynamic arithmetic that checks dimensions at run time,
  the relative comparison, and the ten crossings from a typed quantity.
- `unconv` — the thirteen unit tables, 24 SI prefixes with
  longest-prefix resolution, the absolute and delta conversions, the
  nine typed conversions, and `check_unit` over a table.
- `untext` — the parser for the grammar in the module header, the
  identity of parse-then-format, the cursor-output path, and the five
  per-type formatters.
- `unfault` — the thirteen `message()` arms and the three helpers over
  them.
- `tests/units_edge_tests.nv` — seven cases written after the mutation
  battery, each pinning a line that could be broken with all seven
  suites green: the exponent of one in `undim.is_base`, the dimension
  guard in `untext.format` (a spelling that contradicts the quantity's
  dimension is dropped), the seven-field conjunction in `undim.dim_eq`,
  the number slot `untext.format_bound` adds to the text, the inclusive
  end of `unqty.length_near`, the dimensionless case of
  `undim.has_static_type`, the absolute reading of `unconv.dyn_in`, and
  the `want`-before-`got` order of `unfault.UnWrongDimension` in the
  builder, in `length_from` and through the shared `typed_in` path —
  which last one the battery found still free after the first seven
  cases had landed, and which a case added to the same test now pins.

### Changed

- `src/unqty.nv` — the comment above the private `order` helper
  corrected.  It claimed a two-branch `if` would answer `Equal` for a
  `NaN` "which is the one answer that is certainly wrong"; the helper's
  own three-branch body answers `Equal` for a `NaN` too, because
  `Ordering` has no fourth variant to answer with.  The comment now
  states what the body does and tells a caller holding a possible `NaN`
  to ask `math.is_nan`.  No behaviour changed.
- `tests/*_tests.nv` — the "RED ON PURPOSE" stamp removed from the six
  oracle suites.  The assertions are unchanged.
- `tests/unconv_tests.nv`, `tests/undim_tests.nv`, `tests/undyn_tests.nv`
  and `tests/unqty_tests.nv` — the twenty-one `x == false` sites those
  four pre-existing suites already carried rewritten as `not x`
  (`redundant-bool`), which cannot change a predicate.
- `src/unconv.nv`, `src/undyn.nv`, `src/untext.nv` and
  `tests/units_edge_tests.nv` — the thirty `x == false` sites in this
  release's own new bodies and coverage suite rewritten the same way
  before the lint gate was cleared.  `novo.lint` now answers zero
  findings for all six modules under `src/` and all seven files under
  `tests/`.
- `src/unconv.nv` — `split_prefix` resolves a bare unit before any
  prefix is tried, so `min` is the minute and not a milli-inch, and
  `mdeg` is refused as the prefix `m` over `deg` (which takes none)
  rather than as an unknown unit.  The `UnUnknownPrefix` refusal was
  previously unreachable from a parse.
- `tests/units_edge_tests.nv` — **new.**  The coverage suite for the
  paths the six oracle suites do not reach: both readings of a
  temperature unit — `to_si_absolute` applying the 273.15 offset and
  `to_si_delta` not, asserted side by side so a difference cannot go
  wrong by the offset in silence — plus the exponent forms of the
  grammar, the whole parse-convert-format-parse chain on a magnitude
  that does not divide evenly in the target unit (`3.5 km/h` in `m/s`)
  and on a negative one, both ends of the significant-digit rounder,
  the prefix refusals, the coherent-unit fallback for a dimension with
  no table, both branches of `require_or`, `UnNoStaticType` built by
  hand because no function answers one, and `check_unit` over a unit a
  caller built badly.
- `README.md` — the implementation-status table and the two paragraphs
  that described an unimplemented release deleted; the test count in
  the Tests section corrected to the 17 the suite holds; the "six
  test files" sentence corrected to the seven under `tests/`; and the
  edge suite's count is the 27 it holds after the mutation pins.

### Known

- **Three lines under `src/` are excused by `cov: skip`, each with its
  reason in place**: `unconv`'s unreachable `Err` in `scaled` (a rest
  `split_prefix` has already resolved), `untext`'s out-of-range
  exponent arm (every call site counts 0..6) and a `break` a trimmed
  body cannot reach.  Everything else under `src/` is covered: 99% of
  772 instrumented lines, over the seven suites.
- **`UnDim` is boxed, and it should not have to be.**  A `@value`
  struct may not be a field of a boxed struct nor an enum payload, so
  the representation that fits seven small integers is unavailable
  exactly where it would pay.  Measured, not assumed: it is why
  `unqty` is the only module whose types are unboxed.
- **An offset unit must stand alone in a unit expression.**  `degC/s`
  and `degC^2` are refused (`UnOffsetUnitInExpression`,
  `UnOffsetUnitWithExponent`) rather than resolved, which is the
  package's answer to the temperature trap and costs the `°C/min` rate
  spelling — `K/s` is the unambiguous one.
- **`untext.format` renders a unit from its DIMENSION when the format
  asks for superscripts**, so a `printed_format` quantity prints
  `0.972 km·s⁻¹` where the machine format prints the spelling it was
  parsed from.  Only the machine format round-trips, and parsing a
  superscript form back is not supported.
- **Seven of the twenty-three types publish a dimension accessor.**
  `unqty.length_dim`, `.mass_dim`, `.time_dim`, `.velocity_dim`,
  `.force_dim`, `.energy_dim` and `.pressure_dim` are the seven; the
  three other types `undyn` crosses — temperature, acceleration and
  power — have none, so those crossings call `undim.temperature()`,
  `.acceleration()` and `.power()` directly.
- **`undyn.root` of a negative even root answers a NaN**, not a fault —
  the dimension is what that call refuses.
- **No `@tier(embedded)` claim**, though the typed half would fit one;
  see the README's "What is not included".

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
