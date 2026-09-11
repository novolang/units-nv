# units-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Dimensional analysis in two halves. A named set of `@value` quantity
types the **type checker** enforces — a `UnLength` is not a `UnTime`,
and a call that confuses them does not compile — and a dynamic quantity
with a seven-exponent dimension vector for everything the static set
does not cover.

- `undim` — a dimension as seven exponents, and the algebra;
- `unqty` — the typed quantities, and the arithmetic between them;
- `undyn` — the dynamic quantity, and the bridge;
- `unconv` — units, their tables, and the temperature offset;
- `untext` — reading `3.5 km/h` and writing it back;
- `unfault` — the things that can fail, which are all in the dynamic
  half.

```
novo pkg add units-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use unqty

// A speed from a distance and an elapsed time. Swap the arguments and
// this does not compile.
fn average_speed(distance: UnLength, elapsed: UnTime) -> UnVelocity
    unqty.velocity_of(distance, elapsed)
```

## What the type checker enforces, exactly

The plan's row for this package is *"dimensional analysis the type
checker enforces"*, and being precise about that is the README's first
job, because one thing is true and one is not.

**True.** A function that wants a `UnTime` cannot be given a
`UnLength`:

```
E2001: argument 2 of 'unqty.velocity_of': expected UnTime but got UnLength
```

No test was involved, no review, no run-time check. Every unit
confusion that has ever cost anybody a spacecraft is of this shape, and
this is the whole of what the static half buys.

**Not true.** There is no way to write `a / b` and have the checker work
out that a length over a time is a velocity. **novo-lang has no
arithmetic operator overloading at all**: `+`, `-`, `*`, `/` and `%` are
built-ins over the numeric scalars and are never reinterpreted as a
method call, so

```
E2000: operator '+' not applicable to UnForce and UnForce
```

— on a boxed struct as much as on a `@value` one. Only the **six
comparison operators** dispatch to a trait (SPEC § 3.8: `==` is `eq`,
`<` is `lt`, and so on), and on a `@value` type even those are refused:

```
E2015: a comparison operator on a @value type (UnLength) is not
       supported in v1 — compare the fields you mean (SPEC §14.5)
```

So every operation here is a **named function**, including equality.
`velocity_of(distance, elapsed)` reads better than `distance / elapsed`
would have and `force_of(m, a)` better than `m * a`; what is lost is the
chained expression, and a caller computes in two lines where one would
do. The cost is counted rather than hidden: one `add`, one `eq` and one
`si` accessor **per type**, because there is no trait a one-field
`@value` struct could carry that a generic function could reach through.

## The load-bearing interface: `UnDim`

```novo ignore
pub struct UnDim
    length: Int
    mass: Int
    time: Int
    current: Int
    temperature: Int
    amount: Int
    luminous: Int
```

The SI has seven base dimensions and every physical quantity is a
product of integer powers of them. So a dimension **is** seven integers,
dimensional analysis **is** adding and subtracting those vectors, and
`UnDim` is that value.

It is load-bearing because it is the one thing both halves share. The
static types have a fixed dimension each, which the checker enforces;
the dynamic quantity carries its own, which `undyn` checks at run time;
and every function that crosses between them compares two `UnDim`s.
`unqty.velocity_dim()` is the same fact the checker already knew,
written down.

**And it is boxed, which it should not have to be.** Seven small
integers is exactly what `@value` is for — eight bytes, copied in a
register, no refcount — and this type cannot be one, because a `@value`
struct may not be a field of a boxed struct nor the payload of an enum
or a `Result`. `UnDyn` is boxed (it is what a fallible parse answers)
and holds a dimension; `UnFault` carries two of them in its mismatch
variant. The representation that fits the data is unavailable at the
one place it would pay.

## Where the line between static and dynamic actually falls

It is not a matter of taste. It is drawn by what the unboxed
representation can hold:

> A `@value` struct may not be the payload of a `Result`, an optional or
> an enum, nor a field of a boxed struct.

A parse can fail, so what a parse answers must be boxed. A quantity of
an arbitrary dimension must carry that dimension, and a struct that
carries one is a struct with a field. So:

| | |
| --- | --- |
| **static** — `unqty` | one `Float` each, `@value`, never carried, never compared with an operator, **never returned fallibly**. Every function in `unqty` is total: there is no `Result` in the module at all. |
| **dynamic** — `undyn` | a magnitude, a `UnDim` and the spelling it was parsed from. Boxed, fallible, ordinary. |

A typed quantity library whose arithmetic answered `Result` would be
admitting that its types check nothing. That asymmetry is the whole
argument, and the fault list is where it shows: every variant of
`UnFault` belongs to the dynamic half.

The crossing point pays for the rule visibly. `undyn.require` **cannot**
answer a `Result<UnLength, UnFault>` — the payload rule again — so it
answers a raw magnitude and the caller applies the total constructor:

```novo ignore
let d = untext.parse("3.5 km/h")!
let v = unqty.velocity_mps(undyn.require(d, undim.velocity())!)
```

Two calls where one would read better, and the second cannot fail. It is
the shape `docs/publishing.md` calls *"a fallible codec returns raws"*.

## The static set is a set, not a parameter

`Quantity<Dim>` generic over the dimension is what uom does in Rust with
const generics. novo-lang has neither const generics nor a numeric bound
a `@value` struct could carry, so the static half is a **named list** —
the seven bases, sixteen derived quantities, and two temperatures — and
`undyn` covers everything off it.

What that costs is visible and worth stating: adding a quantity to the
static set is a type, a constructor, an accessor, an `eq`, a `dim` and
whatever arithmetic relates it to its neighbours. What it buys is that
the eighteen quantities a program actually uses are checked at compile
time rather than at run time.

## The temperature offset, which is what this package exists for

**`UnTemperature` and `UnTempDelta` are two types.**

`20 °C` as an absolute temperature is 293.15 K. A temperature
*difference* of 20 °C is 20 K. One type cannot be both: converting an
absolute applies the offset and converting a difference must not, and a
library with one temperature type gets one of them wrong by 273.15 every
time — silently, in the direction nobody checks.

So:

| written | meant | in kelvin |
| --- | --- | --- |
| `to_si_absolute(20, °C)` | a temperature | `293.15` |
| `to_si_delta(20, °C)` | a difference | `20` |
| `to_si_absolute(9, °F)` | a temperature | `260.372…` |
| `to_si_delta(9, °F)` | a difference | `5` |

Two functions rather than a flag, because the choice is the caller's and
a flag is a thing a caller passes wrongly once and never notices.

Three consequences follow, and all three are in the surface:

- `temperature_sub(a, b)` answers a **delta**; `temperature_shift(a, d)`
  takes one. There is no `temperature_add` and no `temperature_scale`,
  because adding two absolute temperatures is not an operation — 20 °C
  plus 20 °C is not 40 °C in any system — and their absence is the type
  system doing the work.
- **An offset unit in a compound expression is refused.** `°C/s` could
  mean a rate of change, where the offset must not apply, or it could be
  a mistake. pint resolves these with rules most of its users do not
  know they rely on; this answers `UnOffsetUnitInExpression` and makes
  the caller write `K/s`, which is unambiguous and the same number.
- An offset unit with an exponent is refused outright, because it has no
  meaning at all.

## Reading and writing `3.5 km/h`

The grammar, whole, because it is small enough to state:

```
quantity   ::= number ws* unit-expr?
unit-expr  ::= factor ( ( '*' | '·' | ' ' | '/' ) factor )*
factor     ::= symbol ( '^' sign? digit+ )?
symbol     ::= prefix? unit-symbol
```

Four things about it are decisions: a **space inside a unit expression
multiplies** (`N m` is a newton-metre, which is how SI writes it); `/`
is **left-associative** (`m/s/s` is `m·s⁻²`, and `m/s^2` is the spelling
to prefer); an exponent **binds to its factor only** (`m/s^2` is not
`(m/s)²`); and a **missing unit is a dimensionless quantity**, not an
error, so a configuration file whose values are sometimes bare reads
with one call — with `parse_required` as the strict door.

**Parse then format is the identity.** `UnDyn` carries the expression it
was written in, so `format(parse("3.5 km/h")!)` is `3.5 km/h` and not
`0.9722222 m/s`. That is what lets a tool read a configuration file,
change one value and hand a person a one-line diff — the same argument
ini-nv's document form makes for comments and blank lines.

**Formatting is not localised, on purpose.** The decimal separator is a
point and there is no group separator, because this is the
machine-readable spelling: it is what `parse` reads back, what goes in a
configuration file and what goes on a wire. A caller showing a number to
a person formats the `Float` with numfmt-nv and writes the unit beside
it — a decision about an audience, which no units library can make.

And **no unit is chosen for the caller**. There is no "pick a readable
unit" heuristic, because `0.000001 m` is right in a CAD file and `1 µm`
is right in a report and no library knows which it is in.

## The tables, and what is deliberately not in them

The table is this package's, not a caller's: `unit_named` resolves
against a fixed set, and a caller with a barn or a survey foot builds a
`UnUnit` with `custom_unit` and passes it. A library with a **mutable
registry** has one module registering a unit and another that does not
know about it — a bug that only appears in the second program to use
both.

Three entries are ambiguous in the world and unambiguous here, because
guessing is how a recipe scales wrongly:

| the world writes | this package refuses, and offers |
| --- | --- |
| `gal` | `gal_us` and `gal_imp`, which differ by a fifth |
| `hp` | metric and mechanical, 1.4% apart |
| `cal` | the thermochemical calorie; the dietary Calorie is `kcal` |

Not units at all, each for a reason:

- **a month and a year.** Neither is a fixed number of seconds; a
  calendar month is 28 to 31 days. That arithmetic is calendar-nv's,
  over dates, and a units library answering "a month is 30 days" is
  answering a question nobody asked.
- **`KiB`, `MiB`.** IEC 80000-13, over a count of bits, which is not an
  SI dimension. A library that accepted them would be answering a
  dimensionless number with a factor of 1024.
- **an instant.** `UnTime` is an *amount* of time. A point in time is
  calendar-nv's or `std.time`'s, and a units library that conflated them
  would let a caller add two dates.

The kilogram is the SI base rather than the gram, so `g` has a factor of
0.001 and `kg` of 1 — the one place the system is inconsistent with its
own prefix rules, and every units library has a special case for it.

## An angle is dimensionless, and that is why it is a type

A radian is `length/length` and cancels, so `UnAngle`'s dimension is the
dimensionless one — which is why `sin` takes a number and why a library
that made the angle a base dimension disagrees with every physics text.

The same shape appears once more: a **torque** and an **energy** have
the same dimension and are different quantities. `work_of` answers a
`UnEnergy`; a torque would need its own type rather than an annotation,
and it is not in the static set. Both cases are the argument for the
static half in one line — the dimension cannot tell them apart and the
type can.

## The layer, and the device claim that is not made

`core` — no effects. Every function here is arithmetic on floats and
integers, and the one thing a units library might reach a host for (a
locale's decimal separator) is not here.

**A `@tier(embedded)` claim would fit the static half, and it is not
made.** `unqty`'s quantities are `@value` structs of one `Float`, and a
device computing a velocity from a distance and a time uses exactly
those — no allocation, no refcount, register-passed. What stops the
claim is that `undim`, `undyn`, `unconv` and `untext` all carry strings
and growable lists, so the probe would have to cover `unqty` alone —
and `host_modules` cannot express that, because it **widens** a module's
budget rather than narrowing a probe's reach. An embedded split would be
a second package (`units-core-nv`: `unqty` plus the seven dimension
constants) and it is named here rather than claimed. The audit's
`core-embedded` row passes as *makes no device claim*.

**No dependencies**, and that is the shape of the package. A units
library's temptation is a formatting package for its output and a parser
for its input; neither pays here, because the spelling is a number, a
space and a unit expression over `*`, `/` and `^` — a grammar small
enough to state in one comment, and small enough that a general parser
would be a dependency for forty lines.

## Where the names come from

Every public type and every enum variant is prefixed `Un`. `Length`,
`Mass`, `Time`, `Dim`, `Unit` and `Quantity` are all names other
packages want, and struct identity is keyed by name across a whole
program. unicode-nv uses `Uni`, which is why this one does not.

## The reference implementation

pint for the API shape — the dynamic quantity, the dimension vector, the
registry-of-units idea and its offset-unit rules — and uom for the
typed half, which in Rust is a `Quantity<Dim, Unit, V>` generic this
language cannot express. The constants in `tests/` are the SI brochure's
and the defining constants of the 2019 redefinition.

Left out and named above: a mutable registry, localised formatting, a
unit-picking heuristic, binary prefixes, calendar durations, torque, and
uncertainty propagation.

## Status

Interface only. `novo pkg build` is clean, `novo doc` renders, the API
tests under `tests/` are red against `todo()` bodies, and the three
shard rows — `effect-budget`, `dep-layer`, `no-discharge-in-core` — are
green. The first implementation is the `0.1.0` published over this.
