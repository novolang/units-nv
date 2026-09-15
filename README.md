# units-nv

Dimensional analysis is the practice of tracking, alongside a number,
which physical quantity it measures, so that a length is never added to
a time and a value in feet is never read as a value in metres. This
package brings it to novo-lang, over the International System of Units
defined in the
[SI Brochure, 9th edition](https://www.bipm.org/en/publications/si-brochure).
Its references for the interface are the Python library
[pint](https://pint.readthedocs.io/) and the Rust crate
[uom](https://docs.rs/uom).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What a dimension and a quantity are

The SI has seven **base dimensions**: length, mass, time, electric
current, thermodynamic temperature, amount of substance and luminous
intensity (SI Brochure section 2.3.1). Every physical quantity is a
product of integer powers of those seven. A velocity is a length divided
by a time, so its exponents are 1 for length and −1 for time and 0 for
the rest. A **dimension** is therefore seven integers, and multiplying
two quantities adds those integers. `UnDim` is that value.

A **quantity** is a number together with a dimension. A **unit** is a
named scale on a dimension: a factor that converts a value into the SI
base for that dimension, and, for the temperature scales, an offset as
well.

This package holds a quantity in two different ways.

A **typed quantity** is one of twenty-three value types, one per
quantity the package names: `UnLength`, `UnTime`, `UnVelocity` and so
on. Each holds a single number in the SI unit for its dimension. The
compiler refuses a call that passes one where another is wanted, so a
unit confusion is a compile error rather than a wrong answer.

A **dynamic quantity**, `UnDyn`, holds a number, a `UnDim` and the
spelling it was read from. It covers every quantity the typed set does
not name, and it is what a parse answers, because a parse can fail.

Two of the twenty-three types are temperatures, and they are separate on
purpose. `UnTemperature` is a point on the scale and `UnTempDelta` is a
difference between two points. Twenty degrees Celsius as a temperature
is 293.15 kelvin; twenty degrees Celsius as a difference is 20 kelvin.
See "The rules a user needs", rule 5.

## Install

```
novo pkg add units-nv
```

## Example

```novo
use unqty
use undim
use undyn
use untext

fn main() [io]
    // A distance and an elapsed time, each in a type of its own.
    let distance = unqty.length_m(1500.0)
    let elapsed = unqty.time_s(310.0)

    // Swapping these two arguments is a compile error, not a wrong answer.
    let speed = unqty.velocity_of(distance, elapsed)
    println("${unqty.velocity_si(speed)} metres per second")

    // Reading a quantity written as text. A parse can refuse.
    match untext.parse("3.5 km/h")
        Err(e) => println(e.message())
        Ok(q) =>
            // The spelling survives the parse, so this prints "3.5 km/h" back.
            println(untext.format(q, untext.machine_format()))

            // `require` checks the dimension and answers the SI magnitude.
            match undyn.require(q, undim.velocity())
                Err(e)  => println(e.message())
                Ok(mps) => println("${mps} metres per second")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: units-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `undim` | A dimension as seven integer exponents, the named dimensions, and the algebra over them: multiply, divide, raise to a power, take a root, compare, and render as text. |
| `unqty` | The twenty-three typed quantities, their constructors and accessors, the physical relations between them, the arithmetic within each, and the temperature operations. |
| `undyn` | The dynamic quantity: construction, arithmetic, comparison, the check that a quantity has the dimension a caller wants, and the conversion from each typed quantity. |
| `unconv` | Units and prefixes: the named table, a custom unit, the conversion to and from SI with and without an offset, and the unit lists per dimension. |
| `untext` | Reading `3.5 km/h` and writing it back, the format options, and the per-type formatting functions. |
| `unfault` | Every reason a dynamic operation refuses, as one enum with thirteen variants. |

## How to choose an entry point

**Use `unqty` when you know at the time you write the code which
quantity you have.** That is most program code. The compiler then
enforces the distinction for free, and every function in the module is
total: there is no `Result` anywhere in `unqty`.

**Use `undyn` when the quantity arrives at run time.** A configuration
file, a command line or a sensor description can name any quantity, so
what reads it must carry the dimension as data. `undyn.require` crosses
back to the typed half: it checks the dimension and answers the plain
number, which the caller hands to the matching `unqty` constructor.

```novo ignore
let q = untext.parse("3.5 km/h")!
let v = unqty.velocity_mps(undyn.require(q, undim.velocity())!)
```

`require` answers a number rather than a `UnVelocity` because a value
type may not be the payload of a `Result`. The constructor that follows
it cannot fail.

**Use `unconv` to convert between units of one dimension** and to look
up a unit by its symbol. **Use `untext` to read and write a quantity as
text.**

## The rules a user needs

1. **Every operation is a named function, including equality.**
   novo-lang does not overload the arithmetic operators, so `a + b` on
   two `UnForce` values does not compile. On a value type even the
   comparison operators are refused (SPEC section 14.5). Write
   `unqty.force_add(a, b)` and `unqty.length_eq(a, b)`. A calculation
   takes two lines where an overloaded operator would take one.
2. **The typed half cannot infer a dimension from an expression.** There
   is no way to write a length divided by a time and have the compiler
   conclude that the answer is a velocity. `unqty.velocity_of(distance,
   elapsed)` is that calculation, named.
3. **What the compiler does enforce is the argument type.** Passing a
   `UnLength` where a `UnTime` is wanted is error `E2001`, at the call,
   with both type names in the message. Every unit confusion that has
   cost anybody a spacecraft has that shape.
4. **A quantity the typed set does not name is a `UnDyn`.** The typed
   set is a fixed list of twenty-three, because novo-lang has no way to
   make a type generic over a dimension. Torque is the case worth
   knowing: it has the same dimension as energy and is a different
   quantity, so `unqty.work_of` answers a `UnEnergy` and a torque has no
   type here.
5. **A temperature and a temperature difference convert differently.**
   `unconv.to_si_absolute` applies the scale's offset and
   `unconv.to_si_delta` does not. The degree Celsius is defined by
   `t/°C = T/K − 273.15`, so the two answers differ by 273.15 every
   time.

   | Written | Meant | In kelvin |
   | --- | --- | --- |
   | `to_si_absolute(20, °C)` | a temperature | 293.15 |
   | `to_si_delta(20, °C)` | a difference | 20 |
   | `to_si_absolute(9, °F)` | a temperature | 260.372… |
   | `to_si_delta(9, °F)` | a difference | 5 |

6. **There is no `temperature_add` and no `temperature_scale`.** Adding
   two points on a temperature scale is not an operation.
   `unqty.temperature_sub` answers a `UnTempDelta`, and
   `unqty.temperature_shift` moves a temperature by one.
7. **A unit with an offset is refused inside a compound expression.**
   `°C/s` could mean a rate of change, where the offset must not apply,
   or it could be a mistake. The refusal is
   `UnOffsetUnitInExpression`, and `K/s` is the unambiguous spelling for
   the same number. An offset unit with an exponent is refused outright
   as `UnOffsetUnitWithExponent`.
8. **A space inside a unit expression multiplies.** `N m` is a
   newton-metre, which is how the SI writes it. The grammar, whole:

   ```
   quantity   ::= number ws* unit-expr?
   unit-expr  ::= factor ( ( '*' | '·' | ' ' | '/' ) factor )*
   factor     ::= symbol ( '^' sign? digit+ )?
   symbol     ::= prefix? unit-symbol
   ```

9. **Division in a unit expression is left-associative and an exponent
   binds to its own factor.** `m/s/s` is metres per second squared, and
   `m/s^2` is the same thing and the spelling to prefer. `m/s^2` is not
   the square of `m/s`.
10. **A quantity written with no unit is dimensionless, not an error.**
    `untext.parse_required` is the strict door for a caller that wants a
    unit every time.
11. **Parse then format is the identity.** A `UnDyn` carries the
    expression it was read from, so formatting `3.5 km/h` gives back
    `3.5 km/h` and not `0.9722222 m/s`. That is what lets a tool change
    one value in a configuration file and produce a one-line difference.
12. **Formatting is not localised.** The decimal separator is a point
    and there is no group separator, because this spelling is what
    `parse` reads back. A program showing a number to a person formats
    the plain number itself and writes the unit beside it.
13. **No unit is chosen for the caller.** There is no heuristic that
    turns 0.000001 metres into 1 micrometre. Which of the two is right
    depends on the document, and no library knows that.
14. **Three symbols are ambiguous in the world, and this package refuses
    them by name.**

    | Written | Refused, and offered instead |
    | --- | --- |
    | `gal` | `gal_us` and `gal_imp`, which differ by about a fifth |
    | `hp` | the metric and the mechanical horsepower, 1.4 per cent apart |
    | `cal` | the thermochemical calorie; the dietary Calorie is `kcal` |

15. **The kilogram is the SI base unit of mass, not the gram.** So `kg`
    has a factor of 1 and `g` a factor of 0.001. This is the one place
    the SI is inconsistent with its own prefix rules (SI Brochure
    chapter 3), and every units library has a special case for it.
16. **An angle is dimensionless.** A radian is a length divided by a
    length. `UnAngle` exists as a type so that a program can say it has
    one, but `undim.is_dimensionless` answers true for its dimension.
17. **The unit table is fixed.** `unconv.unit_named` resolves against
    it. A caller with a unit that is not in it builds one with
    `unconv.custom_unit` and passes it. There is no registry a module
    can add to behind another module's back.

## What is not included

- **Months and years as units of time.** Neither is a fixed number of
  seconds; a calendar month is 28 to 31 days. That arithmetic belongs to
  a calendar, over dates.
- **`KiB`, `MiB` and the other binary prefixes.** They are IEC 80000-13,
  over a count of bits, which is not an SI dimension. Accepting them
  would mean answering a dimensionless number with a factor of 1024.
- **An instant in time.** `UnTime` is an amount of time. A point in time
  belongs to `std.time` or to a calendar package, and conflating the two
  would let a caller add two dates.
- **A mutable unit registry.** See rule 17.
- **Localised number formatting.** See rule 12.
- **A readable-unit heuristic.** See rule 13.
- **Torque, and uncertainty propagation.** Torque needs a type of its
  own rather than an annotation, and it is not in the set. An
  uncertainty is a second number per quantity and a different subject.
- **A microcontroller build.** The typed half would fit: a `UnLength` is one
  float in a value type, with no allocation and no reference count. The other
  four modules carry strings and growable lists, so a probe would have to
  reach `unqty` alone, and a package cannot narrow such a claim to one of
  its modules. An allocation-free subset would be a second package holding
  `unqty` and the dimension constants. Nothing here is claimed to build for
  a device with no heap allocator, and there is no
  `tests/embedded_probe.nv`.

## Related packages

- `std.time` in the standard library measures elapsed time and reads the
  wall clock. It answers instants and durations of a clock, not an
  amount of time as a physical quantity.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is civil
  dates and times. A month and a year live there, where they have a
  calendar to be counted against.
- [numfmt-nv](https://novo-lang.org/packages/numfmt-nv) formats a number
  for a person, with a locale's separators. Take it for the number and
  write this package's unit symbol beside it.

## Tests

```bash
novo test tests/unqty_tests.nv       # 11 tests: the arithmetic, the offset, the tables
```

The constants are the SI Brochure's, including the defining constants of
the 2019 redefinition. The interface follows pint for the dynamic
quantity, the dimension vector and the offset-unit rules, and uom for
the typed half.

The cases that matter most cannot be in the suite. A call with the
arguments swapped is a compile error, and a test file has to compile, so
the refusals are stated in "The rules a user needs" with the error code
each produces. What the suite does assert is the arithmetic those types
carry, that the offset applies to an absolute temperature and not to a
difference, that parse followed by format is the identity, and that
every published unit table is internally consistent and free of
duplicate symbols.

The tests compile today and fail at run, each on the
`not implemented: units-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `undim.UnDim`, `unconv.UnUnit`, `.UnPrefix`, `undyn.UnDyn`, `untext.UnFormat`, `unfault.UnFault` | declared |
| The twenty-three quantity types in `unqty` | declared |
| `undim.scalar`, `.dim`, the seven base dimensions and the fourteen derived ones | no |
| `undim.dim_multiply`, `.dim_divide`, `.dim_power`, `.dim_root`, `.dim_eq`, the four predicates, the three renderings, `.exponent_at` and `.base_symbols` | no |
| `unqty`'s twenty-three constructors and sixteen accessors | no |
| `unqty`'s eighteen physical relations, from `velocity_of` to `angle_of` | no |
| `unqty`'s per-type arithmetic, comparisons and the four temperature operations | no |
| `unqty`'s seven dimension accessors and `static_type_count` | no |
| `undyn`'s construction, inspection, arithmetic, comparison and `require` | no |
| `undyn`'s ten conversions from a typed quantity, and the two static-type queries | no |
| `unconv.unit_named`, `.custom_unit`, `.base_unit`, `.has_offset`, `.split_prefix`, `.si_prefixes`, `.no_prefix` | no |
| `unconv.to_si_absolute`, `.to_si_delta`, `.from_si_absolute`, `.from_si_delta`, `.convert`, `.dyn_in`, `.dyn_of` | no |
| `unconv`'s thirteen unit tables, `.units_of`, `.check_unit`, `.duplicate_symbols` | no |
| `unconv`'s nine per-type conversions | no |
| `untext.parse`, `.parse_required`, `.parse_as`, `.parse_unit`, `.parse_dim`, `.split_text` | no |
| `untext.format`, `.format_in`, `.format_into`, `.format_bound`, `.format_unit`, and the five per-type formats | no |
| `untext.machine_format`, `.printed_format`, `.round_trips`, `.is_quantity` | no |
| `unfault.wrong_dimension`, `.is_dimensional`, `.offset_of`, and the `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
