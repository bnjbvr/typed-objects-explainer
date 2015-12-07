# Class syntax

This is just a sketch, but the idea is to extend class type syntax to
integrate with typed objects.

*WARNING:* This is *very* preliminary and not fully thought through.

## Struct classes

Because `struct` and `value` aren't reserved words, they can't be used as the
keyword for declarative definition of struct and value types, respectively.
Instead, they're prepended to the `class` keyword for declarative type
definitions, which are mostly syntactic sugar for the imperative form.

The equivalence isn't perfect, though: declaratively added methods are
non-generic, and the declarative syntax enables sub-classing. See consecutive
sections for details.

Thus:

```js
struct class PointType {
  x: float64;
  y: float64;

  delta() {
    return this.y / this.x;
  }
}

value class ColorType {
  r: uint8;
  g: uint8;
  b: uint8;
  a: uint8;

  [Symbol.TypeName]: Symbol("ColorType");

  opacity() {
    return this.a / 0xff;
  }
}

// Roughly equivalent imperative definitions:
const PointType = new StructType({x: float64, y: float64});
PointType.prototype.delta = function() {
  return this.y / this.x;
}

const ColorType = ValueType(Symbol("ColorType"),
                            {r: uint8, g: uint8, b: uint8, a: uint8});
ColorType.prototype.opacity = function() {
  return this.a / 0xff;
}
```

Note how neither `PointType` nor `ColorType` have constructors: the instances
are initialized from `source objects`, just as imperatively created struct and
value type definitions.

For struct types, the reason is mostly consistency: how to use struct types
shouldn't depend on how they're defined. The declarative form also wouldn't be
pure sugar for the imperative one, anymore.

For value types, there really isn't any alternative: since they don't have
nominal identity, constructing them doesn't make sense, and since they're
immutable, they must be fully initialized upon instantiation, either ruling out
a free-form imperative initializer, requiring runtime checks for its validity,
or a two-step initialization where all fields are initialized to default values
first.

Further, note the `[Symbol.TypeName]` on `value class ColorType`: this is
equivalent to providing a symbol as the first argument to the `ValueType`
function.

A downside to this scheme is that it requires a no-newline restriction within
`struct class` and `value class`.

### Typed Object Array methods

Since each typed object definition has an accompanying definition of an array
type of instances of the typed object definition, it should be possible to add
methods to that array type declaratively.

This should be done in a similar fashion to how static methods are added to
normal classes (or indeed struct or value type classes). Since that hasn't
been fully defined yet, this section will be fleshed out later.

## Method Non-genericity

Methods of declaratively-defined struct and value objects aren't generic:
calling them on a receiver that's not an instance of the original type
definition or a subclass of it. This adds optimization opportunities as the
engine can omit checks for property accesses within the methods.

## Inheritance

Just as normal classes, value and struct class declarations can use `extend`.
The base has to be a compatible type definition: struct types can only extend
other struct types, value types can only extend other value types.

The memory layout of a child type is simple: the super type's properties form
the pre- and the child type's the postfix of the combined layout. This setup
enables treating child types as instances of their super types.

As an example, the code

```js
struct class RGB {
  r: uint8;
  g: uint8;
  b: uint8;
}

struct class RGBA extends RGB {
  a: uint8;
}

let pixel = new RGBA({r: 00, g: 11, b: 22, a: 44});
```

Results in an object `pixel` with the following layout in memory:

    +==============+    --+ RGBAType
    | r: uint8: 00 |      |
    | g: uint8: 11 |      |
    | b: uint8: 22 |      |
    | a: uint8: 44 |      |
    +==============+    --+

As is the case for the builtin primitives like `String` and `Number`,
struct and value types can only be extended using declarative syntax.

### Type Aliases

By extending a base type without adding any properties, it's possible to
effectively define a type alias with its own set of methods. E.g., `value
class Color extends uint32 {}` creates a `Color` alias for `uint32`.

### Typed Object Array sub-classing

Extending a struct or value type also automatically sets up the child class'
accompanying `array` type as a child class of the parent class' `array` type.

## Sealed classes

It'd also be nice to be able to have classes whose prototypes are
sealed after they are constructed. This gives better optimization
opportunities. It is orthogonal to the syntax above.

```
sealed class Foo { ... }`
```

desugars into something which freezes `Foo.prototype` after
construction.

## Open questions

- How to integrate class syntax with forward references.
- Can we integrate with module loading somehow to avoid the need for
  forward references when using declarative syntax?
