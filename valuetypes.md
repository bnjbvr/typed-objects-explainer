# Explainer for value objects

**Note:** The material in this spec is under discussion.

## Motivation

JavaScript today includes a distinction between value types (number,
string, etc) and object types. Value types have a number of useful
properties:

- they have no identity and compare by value;
- they have no associated mutable state;
- they can be sent across realms;

Unfortunately, the set of value types is currently closed and defined by
the spec. It would be nice if it were user extensible.

There are a number of goals:

- The system should be compatible with the existing value types like
  string, symbol, and number. Ideally it should be a generalization of
  how those value types work rather than its own different thing.
- The engine should be able to freely optimize the representation of
  these new value types. For example, it should be possible to
  duplicate them invisibly to the user and/or store a copy on the
  stack and so forth.
- Value types should integrate with struct types, i.e. it should be possible to
embed value type data into struct type instances.

## The ValueType meta type definition

We introduce the meta type definition `ValueType`. `ValueType` can be
used to define a new value type definition. It takes the same
arguments as `StructType`, with the exception of a mandatory symbol as
the first argument and the lack of an `options` parameter. Unlike `StructType`,
`ValueType` is a regular function, not a constructor, and hence you do not use
the `new` keyword with it (I'll explain why in the section on prototypes below).

```js
const ColorType = ValueType(Symbol("Color"),
                            {r: uint8, g: uint8, b: uint8, a: uint8});
```

The type of every field in a value type must itself be a value type or
else an exception is thrown.

Just as for struct types, creating value array types is done using an overload of the `ValueType` function:

```js
const ColorsListType = ValueType(Symbol("Color"), uint8, 4);
```

Passing a type and number `n` is equivalent to creating a value type with
fields named `0...n-1` and all with the same type, plus a named field `length`
with the value `n`.

## Instantiating value types

You can create an instance of a value type by calling the type definition:

```js
let color = ColorType();
color.b === 0;
```

The result is called a *value object*: it will have the fields specified in
`ColorType`. If no arguments are provided, each field will be initialized to an
appropriate default value based on its type (e.g., numbers are initialized to 0, fields
of type `string` are initialized to the empty string `""`, and so on). Fields with
complex value types are recursively initialized.

Usually a `source object` is provided when when creating a new value type instance.
This object will be used to set the values for each field:

```js
let color = ColorType({r: 22, g: 44, b: 66, a: 88});
color.b === 66;
let color2 = ColorType(color);
color2.b === 66;
```

As the example shows, the source object can be either a normal JS object or another
typed value or struct object. The only requirement is that it have fields of the
appropriate type.

## Assigning to fields of a value type

Values are deeply immutable. Therefore, attempting to assign to a
field of a value throws an error:

```js
let color = ColorType({r: 22, g: 44, b: 66, a: 88});
color.b = 77; // throws
```

## Prototypes

Instances of value types are *not* objects. Therefore, like a number
or a string, they have no associated prototype. Nonetheless, we would
like to be able to attach prototypes to them. To achieve this, we
generalize the wrapper mechanism that is used with the existing value
types like string etc.

The idea is that there is a *per-realm* (not embedding-global!) registry of
value types defined in the engine. This registry is pre-populated with
the engine-defined types like strings and numbers (which map to the
existing constructors `String`, `Number`, etc).

When you perform a property lookup on a value type instance with a property name that's
not a field of the type definition, the value's type definition is looked up in the
per-realm registry. If an entry `C` exists, then a wrapper object is constructed whose
`[[Prototype]]` is `C.prototype` and execution proceeds as normal.

When you invoke the `ValueType` function to define a new type, you are
in fact submitting a new type to be added to the registry, if it does
not already exist.  If an equivalent entry already exists, it is
returned to you. (More details on this process are in the next section.)

All this machinery together basically means you can attach methods to
value types as normal and everything works:

```js
const ColorType = ValueType(Symbol("Color"),
                            {r: uint8, g: uint8, b: uint8, a: uint8});

ColorType.prototype.average = function() {
    return (this.r + this.g + this.b) / 3;
};

let color = ColorType({r: 22, g: 44, b: 66, a: 88});
color.average(); // yields 44
```

## The registry and structural equivalence

I said above that when you invoke the `ValueType` function, you are
in fact submitting a new value type to be registered.  If a
*structurally equivalent* value type definition already exists, then that definition
will be returned to you. This, by the way, is why we do not use the
`new` keyword when creating a value type -- it is not *necessarily* a
new object that is returned.

But when are two types *structurally equivalent*? The answer is
fairly straightforward:

- Every type definition is structurally equivalent to itself.
- Two user-defined value-type definitions are structurally equivalent
  if:
  - they both have the same associated symbol
  - they both have the same field names defined in the same order
  - the types of all their fields are structurally equivalent to one another

Let's go through some examples. First, if you register the same type
definition twice, you get back the same object each time:

```js
// Registering the same type twice yields the same object
// the second time:
let ColorSymbol = Symbol("Color");
const ColorType1 = ValueType(ColorSymbol,
                             {r: uint8, g: uint8, b: uint8, a: uint8});
const ColorType2 = ValueType(ColorSymbol,
                             {r: uint8, g: uint8, b: uint8, a: uint8});
ColorType1 === ColorType2;
```

Note that the symbol is very important. If I try to register two value
types that have the same field names and field types, but different
symbols, then they are *not* considered equivalent (here, assume that
the omitted text `...` is the same both times):

```js
// Different symbols:
const ColorType1 = ValueType(Symbol("Color"), ...);
const ColorType2 = ValueType(Symbol("Color"), ...);
ColorType1 !== ColorType2;
```

Similarly, if the symbols are the same but the field names are not,
the types are not equivalent:

```js
let ColorSymbol = Symbol("Color");
const ColorType1 = ValueType(ColorSymbol,
                             {r: uint8, g: uint8, b: uint8, a: uint8});
const ColorType2 = ValueType(ColorSymbol,
                             {y: uint16, u: uint16, v: uint16});
ColorType1 !== ColorType2;
```

### Design discussion

This setup is designed to permit multiple libraries to be combined
into a single JS app without interfering with one another. By default,
all libraries will create their own symbols and hence their own
distinct sets of value types. This means you can freely add methods to
value types that you define without fear of stomping on somebody
else's value type.

On the other hand, if libraries wish to interoperate, they can do so
via the symbol registry. Similarly, value types that should be
equivalent across realms can be achieved using the global symbol
registry.

Put another way, ES6 introduced symbols in order to give libraries a
flexible means of declaring when they wish to use the same name to
refer to the same thing versus using the same name in different ways.
Value types build on that.

## Comparing values

Two values are consider `===` if their types are structurally
equivalent (as defined above) and their values are (recursively)
`===`. As a simple case of this definition, if I instantiate two
instances of the same value type with the same values, they are `===`:

```js
let color = ColorType({r: 22, g: 44, b: 66, a: 88});
color === ColorType({r: 22, g: 44, b: 66, a: 88});
color !== ColorType({r: 88, g: 66, b: 44, a: 22});
```

In fact, If I define `ColorType` twice, but using the same `Symbol`,
instances are *still* equivalent (here, assuming that the stuff I
omitted with `...` is the same in both cases):

```js
let symbol = Symbol("Color");
const ColorType1 = ValueType(symbol, {...}); // ... == rgba
const ColorType2 = ValueType(symbol, {...});
ColorType1({r:22, ...}) === ColorType2({r: 22, ...}); // ... == same values
```

However, If I define `ColorType` twice, but using different `Symbol`
values, then instances are *not* the same (again, assume that the stuff
I omitted with `...` is the same in both cases):

```js
const ColorType1 = ValueType(Symbol(...), {...}); // ... == rgba
const ColorType2 = ValueType(Symbol(...), {...});
ColorType1({r:22, ...}) !== ColorType2({r: 22, ...}); // ... == same values
```

Note that I have only defined `===` here. It is not clear to me at
this moment what the behavior of `==` ought to be: it might be useful
for `==` to ignore the symbol and only consider field names and types
when deciding whether two types are equivalent. Alternatively, if we
introduce overloadable operators as [described here](overloading.md),
then this could potentially be done in user libraries.

## The typeof operator

The `typeof` operator applied to a value type yields its associated
symbol. This changes `typeof` so that it does not always return a
string. If this is deemed too controversial, then `typeof` could
return the stringification of the associated symbol, but that seems
suboptimal to me because in that case `(typeof v === typeof w)` has no
real meaning (it also permits "forgery" of the return values
`"string"` and so forth). On the other hand, that makes user-defined
value types somewhat inconsistent with the existing value types like
string.
