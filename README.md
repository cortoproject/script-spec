# cortoscript specification
Cortoscript is a new data definition language designed specifically for modeling objects and datamodels that describe the real-world. It differs from other definition languages like XML and JSON in that it:

- is strongly typed
- supports units, ranges and semantics
- supports hierarchies of objects
- supports directed object graphs
- supports inheritance and polymorphism
- supports expressions

This combination of features makes cortoscript a good fit for Internet of Things systems, as these benefit from well-defined interfaces, and interoperability at the semantics level.

While cortoscript is the native definition language for the corto framework (https://corto.io), the language itself does not depend on corto. It makes little assumptions about the underlying type system, which allows it to be easily ported to other frameworks.

The language does not contain a syntax for describing models, though if the underlying type system is self-describing (meaning types can be treated as 'regular' data), the language is capable of creating both models and data. The language syntax has been optimized to make this look and feel natural.

This is a small example of a type and an object of that type in cortoscript, using the corto type system:

```
class Drone {
    lat: float64, tags:[latitude]
    long: float64, tags:[longitude]
    alt: float64, unit:feet, tags:[altitude]
}

Drone my_drone (
    lat: 37.7749,
    long: 122.4194,
    alt: 250ft)
```

This repository contains an up to date parser based on an ANTLR4 grammar:
https://github.com/cortoproject/script-parser

This project transforms the ANTLR datastructures into an AST:
https://github.com/cortoproject/script-ast

This project parses the AST and creates corto objects (in development):
https://github.com/cortoproject/script-declare

## History
Development of cortoscript started in 2012 as part of a project called Fractal. Fractal was a little hobby project intended to make it easier to write games that rely on multiple technology stacks, like Qt, Sigar and OpenSplice DDS.

Fractal featured a strongly typed scripting language, which at the time was built because it seemed like a fun thing to do. This scripting language eventually morphed into cortoscript, and fractal evolved into corto (https://corto.io).

The syntax of the fractal scripting language was quite different from cortoscript, though it already featured strong typing, a self-describing type system and support for object hierarchies. Hierarchies got special attention, as DDS, Qt and game engines all featured hierarchical designs that needed to be wrapped somehow in the fractal framework.

This is an example from 2012 of fractal script. The language used indentation and a scope operator ("::") to indicate object hierarchies:

```
struct Point = {{
  {"x", uint32},
  {"y", uint32}
}} ::
  void add(Point p):
    this.x += p.x;
    this.y += p.y

Point p = {x=10, y=20} ::
  Point q = {10, 20}

p.add(p::q)
```

After a series of name changes because of  confusion with other products ("hyve"), unavailability of domain name ("fractal", "lyra") or trademark conflicts ("cortex"), the project eventually settled on the name corto and went open source in November 2014.

The language also went through a number of iterations. The focus shifted from being a scripting language to a data and model definition language. Features were added to make defining models more intuitive, and organize models in separate packages. The C++ scope identifier ("::") was replaced with the forward slash ("/"). This is an example of a model definition in the corto language:

```
in package my_package

struct Point:/
  x, y: int32

Point x {x=10, y=20}
```

This latest iteration of the language was used in a lot of corto-related projects, and is still in use today (https://github.com/cortoproject/corto-language). While this version worked reasonably well for defining simple models, it had the disadvantage that for complex model- or data definitions, the indentation made scripts difficult to read.

This implementation had several other problems. Like its predecessors it was implemented using lex/yacc which made it fast and gave it a low footprint, but restricted expressiveness of the parser, and made for some very ugly parser code. Additionally, the generated parser did not clean up its state after parsing code, which made it unsuitable for REPL usecases. By 2017, efforts were already underway to rewrite the parser in ANTLR3, but because of shifting priorities this work got stalled many times.

In February 2018, after some particularly bad examples of how obfuscated the indented code could get, the decision was made to overhaul not just the parser, but also the language, and work on cortoscript began. The project picked up speed as the release of the ANTLR4 C++ runtime in 2017 made it possible to switch to ANTLR4, which was another step up in productivity from ANTLR3.

An example of the old corto syntax for a complex type:

```
class register {
    base = member,     
    options = {
        parentType = template,
        parentState = DECLARED
    }
} :/
    member address: uint16
    member sample_frequency: uint16
```

In March 2018 the new language was given the name cortoscript. It was decided to design the language in a way that it would be framework independent so it could be used without corto, and to include a number of features from scratch that would make it more applicable to IoT usecases, like annotating values with their respective units. Significant whitespace (indentation) was dropped, and a new syntax was adopted that resembled functions in C/C++, to give the language a familiar look:

```
Type identifier (value1, value2) {
    // children
}
```

The first corto-based cortoscript parser is still in development. A list of relevant repositories is included in the introduction of the README. If you would like to get involved in cortoscript development, send a message in the message box on the corto website (https://corto.io), or send an email to sander@corto.io.

## Design principles
Cortoscript must be declarative. It must provide syntax to instantiate any kind of data that is allowed by the underlying type system without having to rely in imperative statements, including creating cyclic graphs.

Cortoscript must be usable independently from corto. The language must not contain anything that is specific to the corto type system or runtime.

Cortoscript must be strongly typed, yet must not have a DSL for defining types. Types must be defined like ordinary objects, which requires that the underlying type system is self-describing.

Cortoscript must be concise and easy to read. It must take advantage of the underlying type system to provide a syntax which is less verbose and contains less clutter than alternatives.

Cortoscript must be DRY (don't repeat yourself) by introducing syntax
constructs that allow scripts to avoid repetition.

Cortoscript should be easy to parse. The number of syntax constructs in cortoscript should be in the same order of magnitude as JSON and XML, which should make it a language that can be parsed within reasonable time.

Cortoscript should support internet of things usecases by supporting notation for hierarchical data, as well as support for units, ranges and data semantics.

## Language overview
This section does not aim to provide an exhaustive overview of the language, as
the language grammar is a more appropriate and concise way to do that. Here we
describe the high-level design of the language, so that a reader will be able
to read and write definitions in cortoscript.

Cortoscript is a declarative language, which means that scripts use declarations
to instantiate data. Declarative languages declare the "things that should be",
whereas an imperative language the "things that should be done". Here is a
simple C example that demonstrates the differences between declarative and
imperative styles:

Imperative (C):

```c
Point p;
p.x = 10;
p.y = 20;
```

Declarative (C):

```c
Point p = {.x = 10, .y = 20};
```

The following sections provide an overview of the different parts of the language.

### Whitespaces and line endings
Whitespaces in cortoscript are mostly ignored, except for the newline character
which can be used to separate statements. In addition to the newline character
a script can also use the semicolon (`;`) to separate statements.

Inside parentheses (`()`) and brackets (`[]`) newlines are ignored which makes
it possible to split up complex statements over multiple lines.

There are certain cases where whitespaces between tokens matter, which is a side
effect of the order in which the lexer rules are specified. For example:

```
foo/bar
```

is interpreted as the identifier `foo/bar` whereas

```
foo / bar
```

is interpreted as `foo` divided by `bar`. This intentional enforcing of using
whitespaces to disambiguate one rule from another allows for more intuitive
interpretation of a script. Cortoscript deliberately avoids confusion in
scenarios like the following C++ snippet:

```
Foo ::Bar
```

Here, intuitively it looks like a declaration of type `Foo` and identifier `Bar`
whereas actually this will be parsed as identifier `Foo::Bar`.

### Declarations
In cortoscript, a declaration takes one of the following forms (simplified):

```c++
type identifier: value
type identifier (value)
type identifier [value]
type (value)
```

The notation that uses `:` is referred to as the "shorthand" notation, and
allows object values to be specified on a single line. The notation that uses
`()` is called the "composite notation", and is better suited for complex
declarations that span multiple lines. The `[]` syntax is called the
"collection notation" and is similar to the composite notation, except that it
is used for describing collection values.

Here is an example of a real cortoscript declaration using shorthand notation:

```c++
Point p: 10, 20
```
It instantiates a new object `p` with type `Point` and value `10, 20`. The same
declaration could be written like this:

```c++
Point p (10, 20)
```

Cortoscript allows for explicitly specifying the member names, such that the
above declarations can be rewritten like this:

```c++
Point p: x:10, y:20
Point p (x:10, y:20)
```

Member assignments can also use the composite notation, which is useful when
assigning complex values:

```c++
Line l (
    start: (x:10, y:20),
    stop: (x:30, y:40)
)
```

Member names may refer to nested members, such that the previous example can be rewritten as:

```c++
Line l (
    start.x: 10,
    start.y: 20,
    stop.x: 30,
    stop.y: 40
)
```

The following example demonstrates an object that contains a collection value:

```c++
list(int32) Integers [10, 20, 30]
```

The observant reader will notice that the list type uses a notation that is
consistent with the declaration syntax. More on that later.

Composite, collection and shorthand notations can be mixed in the same
declaration. There are no hard and fast rules for when to apply the shorthand
notation versus the composite/collection notation.

Here is an example declaration where the three styles are combined:

```c++
list(Line) lines [
    (start: (x:10, y:20), stop: (x:30, y:40)),
    (start: (x:50, y:60), stop: (x:70, y:80))
]
```

#### Collection literals
Collection values can be created in cortoscript as shown earlier, like this:

```c++
list(int32) Integers [10, 20, 30]
```

This creates a new `Integers` object which is a list of integers. By leaving out the object identifier, we can create an anonymous list object:

```c++
list(int32)[10, 20, 30]
```

Cortoscript provides a shorthand notation for creating anonymous lists:

```c++
[|int32| 10, 20, 30]
```

This notation guarantees to return a linked list of `int32` elements.

#### Forward decalations
Cortoscript allows for forward declarations of objects. This allows for
declaratively creating objects that contain cycles (see 'References'). An object
can be forward-declared simply by omitting its value:

```c++
// Forward declaration of p
Point p
```

A declared object needs to be defined before its value may be interpreted.
Internally, corto keeps track of the object state to determine whether the
object has already been defined. In cortoscript, an object will be defined
after it is redeclared with a value. The following code illustrates this:

```c++
// Forward declaration of p
Point p

// p is now declared, but not yet defined

// Define p
Point p: 10, 20

// p is now defined
```

#### Default values
Sometimes objects should not receive an initial value that is different from
their default value. In these cases, a user must still provide a value, because
otherwise the statement would be a forward declaration. The following example
shows how to declare an object that will be defined, yet does not set a value:

```c++
Point p () // leave the value to the default value of Point
// p is now defined
```

#### Default types
When a declaration omits a type, the type is determined according to rules of
the type system (the behavior is equivalent to doing a `corto_create` where
`NULL` is provided for the type parameter).

The most common application for this is where a parent object specifies  a
default type for its children. Take the following example:

```c++
ParentType Foo {
    Child1: 10
    Child2: 20
}
```

If in this example `ParentType` would have set its `defaultType` to `ChildType`,
the type of `Child1` and `Child2` will default to `ChildType`.

Note that the exact semantics on what the default type is for a particular
scenario is not defined by cortoscript, but by the underlying typesystem.

### Defining hierarchies
One of the key features of cortoscript is that it has native support for
describing hierarchically organized objects. Each object in cortoscript can act
as a parent for child objects. Child objects are added to the "scope" of a
parent object. In cortoscript, a scope is opened with the `{}` notation. The
following example shows a parent object with two child objects, of which one
is nested two levels deep:

```c++
Point p (10, 20) {
    Point p (70, 80)
    Point q (30, 40) {
        Point r (50, 60)
    }
}
```

As can be seen, multiple objects can be added to a scope, and scopes can be
nested. Each scope opens a new namespace, which means that objects in a scope
may use names from their parent(s). Nested objects can be referenced using the
`/` operator. The `r` object from the previous example can be referred to as
`p/q/r`.

An object identifier can also be nested, like in the following example:

```c++
Point p/q (10, 20)
```

This will create a new object with identifier `q` in the scope of `p`. If the
`p` object does not yet exist, it will be created with an `unknown` type.
Objects of the `unknown` type are replaced with a typed object once the type is
known. This is illustrated in the following example:

```c++
// P is created as unknown
Point p/q (10, 20)

// P is recreated as a Point, while preserving q in its scope
Point p (30, 40)
```

Cortoscript does not mandate how `unknown` objects are stored by the underlying
store. What is important however is the notion of order. `p/q` is defined before
`p`, which means that the corresponding lifecycle hooks and events must take
place in that sequence.

#### Default scope types
In some scenarios, an application will want to create a number of objects that
are all of the same type. In such a case, it would not be very DRY to have to
specify the type multiple times. Cortoscript supports using an implicit type
when the type of the parent object specifies a type, however in some cases the
parent type might not specify a type, or specify the wrong type.

To accommodate for these scenarios without having to introduce a new type that
specifies the desired default type, cortoscript has a syntax that enables
setting the default type for a scope:

```c++
void parent {|Point|
    p: 10, 20
    q: 10, 20
}
```

In this example, both `p` and `q` will be created as instances of `Point`. Note
that `|Point|` is only a syntax construct, and provides a hint to the
interpreter. After a script is parsed, this information is no longer stored
anywhere. It is therefore possible to open the same scope multiple times with
different default types:

```c++
void parent {|Point|
    p: 10, 20
    q: 10, 20
}

void parent {|Line|
    l: (10, 20), (30, 40)
    m: (50, 60), (70, 80)
}
```

Declarations may still provide an explicit type, in which case the explicit
type takes precedence:

```c++
void parent {|Point|
    p: 10, 20
    q: 10, 20
    Line l: (10, 20), (30, 40)
}
```

### Defining types
A unique property of cortoscript is that it does not have a separate DSL for
specifying types. To get an idea for what this means in practice, consider the
following type description:

```c++
struct Point {
    x: int32
    y: int32
}
```

This example describes a struct type with the name `Point`, and two members
called `x` and `y`, both of which have `int32` as their member type.

What is noteworthy here is that this example can be interpreted using the same
syntax as the ones that have been outlined in the previous sections. According
to those rules, this example describes:

- An object with type `struct` and identifier `Point`
- Two child objects with identifier `x` and `y`, value `int32` and implicit type

To make it more obvious what is happening, this is what it looks like when the
example is written out in full:

```c++
struct Point {
    member x: type:int32
    member y: type:int32
}
```

From this we can make a few observations. The implicit type of `x` and `y` was
replaced with the `member` type, in this case because the typesystem mandates
that the default type for children of a `struct` object is `member`.

Additionally, we can see that a `member` is an type that has a `type` field.
Because `type` is the first field in the `member` type, it can be assigned
without explicitly specifying the `type` field name.

Describing types in this manner is possible because the type system is
self-describing. In the example, `struct` and `member` are not keywords of the
language, but instead they are types themselves in the type system. The
structure of those types is self-describing, meaning that at some level, a
`struct` describes itself as something that has a collection of `member`
instances.

More examples of how types are described can be found in the Code examples
section.

### Primitive values
Cortoscript supports a range of primitive values and corresponding
notations. Here is a summary:

| Kind | Example |
|------|---------|
| boolean | true, false |
| character | 'a', '\0' |
| integer | 10, 0, -10 |
| floating point | 10.5, 105e-1 |
| hexadecimal | 0xFF0000 |
| string | "Hello World", null |

Cortoscript does not mandate things like the size of an integer or the maximum
length of a string. These are determined by the underlying storage and type
system. Take the following example:

```
int32 my_int: 15
```

Here, an object called `my_int` is defined with type `int32`, and its value is
set to `15`. The semantics that determine that `my_int` is a 32 bit integer are
derived from the underlying storage the implementation of the `int32` type. To
cortoscript, these details are opaque.


### References
Cortoscript supports objects that refer to other objects. Consider the following
example:

```
object a
object b: a
```

The `object` type in the corto type system specifies a reference to another
object. In the above example, the `b` object refers to the `a` object. A struct
can use the `object` type to include references to any number of other objects:

```c++
class Person {
    spouse: Person
    mother: Person
    father: Person
}
```

In the corto type system, `class` is a so called "reference type". That means
that every instance of `class` is an object (as opposed to an inline value), and
a member of a `Person` type will contain a reference to an instance of `Person`.

This type can be instantiated like this:

```c++
Person my_father // Forward declaration, to allow for father-mother cycle
Person my_mother (spouse: my_father)
Person my_father (spouse: my_mother)
Person me (father: my_father, mother: my_mother)
```

This instantiates the following object graph:

```
my_father <-> my_mother
       ^      ^
        \    /
         \  /
          me
```

There are various rules in the type system that determine whether an object can be assigned to a reference of a specific type. The aforementioned `object` type accepts references of any type, whereas an instance of a `class` type only accepts instances of that class. This is more exhaustively described in type system documentation.

#### By-value or by-reference operators
By default, corto objects are referred to by either reference or value depending on whether they are of a refernce type or value type. In some cases, a user may want to refer to an object of a reference type by value, or vice versa, refer to an object of a value type by reference. For those scenarios, reference operators allow the user to indicate how objects should be accessed.

The following example illustrates how comparing two objects of a value type will automatically do a compare-by-value:

```c++
int32 a: 10
int32 b: 10

a == b // 'true', because a and b have the same value
```

If a user wants to compare by reference, the unary reference operator (`&`) can be used to accomplish this:

```c++
int32 a: 10
int32 b: 10

&a == &b // 'false', because a and b are different objects
```

Similarly, a user may want to compare two objects of a reference type by value. For this, the unary value operator (`*`) can be used:

```c++
class Point {
    x, y: int32
}

Point p, q (10, 20)

p == q // 'false', because p is not the same object as q
*p == *q // 'true', because the value of p is equal to q
```

Using the value operator on an object of a value type, or the reference operator on the object of a reference type is valid, but has no effect.

### Identifier resolution
When referring to other objects in cortoscript, there are certain rules that
govern how those objects are resolved from the underlying store. Understanding
these rules will help preventing what could otherwise be confusing issues.

Identifier resolution is primarily the responsibility of the underlying store.
Cortoscript will use what is provided by the underlying store, but does require
that the mechanism offered meets a number of requirements. These will ensure
that scripts written in cortoscript work predictably across implementations.

#### Case insensitive
Firstly, cortoscript is case insensitive. The first reason for this decision is
because it makes it easier to use as an extension of OMG-IDL, which is widely
adopted in industrial IoT systems as the data modeling language for the OMG DDS
standard.

The second reason is that some file systems are case insensitive. As
cortoscript will often be used as a front-end for code generation, type names
which only differ in case would could cause generated files with names based on
the typename to overwrite.

A third reason is that different contexts may require different case
conventions. A good example is the `modifiers` field of a `member`. This field
is a bitmask that represents different options for member objects, like
`readonly`, `optional` etc. In C code, bitmasks are represented as upper-case
identifiers (`READONLY`), whereas in cortoscript it is more natural to use the
the lowercase variant (`readonly`).

Lastly, it is good practice to not create objects with the same name that only
differ in case.

#### Identifier scoping
When referring to other objects in cortoscript it is not always necessary to
use the full identifier of the object. Identifiers are relative to the scope in
which they are used. To illustrate this, take a look at this code, which
uses the `Person` class from the previous example:

```c++
void my_family {
    Person mother
    Person father: spouse:mother
    Person mother: spouse:father
}
```

Note how the references to `mother` and `father` are relative to the `my_family`
scope. The full identifiers would have been `/my_family/mother` and
`/my_family/father`.

When a reference is not found in the current scope, the parent scope will be
searched recursively, until the root of the store has been reached. Take the
following example:

```c++
void my_family {
    Person grandmother()
    void my_household {
        Person father (mother: grandmother)
    }
}
```

In this scenario, the `grandmother` object can be referred to by its relative
identifier, because the object is located in the `my_family` scope, which is the
parent object of the `my_household` scope.

Note that relative identifiers can only be used from within the scope where they
are used. Take the following example:

```c++
void my_family {
    Person grandmother()
    void my_household()
}
Person my_family/my_household/father (mother: my_family/grandmother)
```

Here, we need to use the identifier relative to the root for `grandmother`, as
the reference is specified outside of the `my_family` scope, even though the
`mother` object is inside the scope.

For objects belonging to the type system, cortoscript mandates that they can be
accessed from anywhere using just the identifier relative to their scope.

To illustrate, in corto the type system is located in the `/corto/lang` scope.
That means that all meta-types, like `struct`, `class` and so forth, are stored
in this scope. It would however be inconvenient if you would have to write:

```
corto/lang/struct Point {
    corto/lang/member x, y: corto/lang/int32
}
```

To ensure convenience, type system objects must therefore always be directly
accessible without specifying a scope. In addition, to ensure that a user
cannot accidentally redefine objects from the type system, the type system
scope should be searched first. Take the following example:

```
struct Struct {
    m: int32
}

struct Point {
    x, y: int32
}
```

Though a contrived scenario, it demonstrates a potential ambiguity, where the
definition of `Point` could use either `/Struct` or `/corto/lang/struct`. Since
the type system scope (`/corto/lang`) is always the first place for a lookup,
this ambiguity cannot occur.

If an application wants to resolve its own definition of a type system object,
it will have to use the full path, which in our example would be `/Struct`.

In addition to the type system path, Corto also adds the `corto` scope to the
default search path. To ensure this does not interfere with resolving regular
identifiers, the corto scope will only be searched when none of the other scopes
yielded a result.

#### Resolving constants
A common kind of type in typesystems is an enumeration, or a bitmask. In a
hierarchical store, a natural way to describe an enumeration is to store the
constants of an enumeration in the scope of the enumeration type. This is
exactly what the corto type system does, and allows for specifying enumerations
like this:

```cpp
enum Color {
    Red
    Green
    Blue
}
```

The `Red`, `Green`, and `Blue` objects are instances of the `constant` type,
which is a 32-bit integer. When assigning an enumeration value, you could
use the constant objects in the initializer, like this:

```cpp
Color my_color: Color/Red
```

Having to specify the enumeration name when referring to a constant however is
not very convenient, especially since it is evident that `Red` should be part of
`Color`. To make these scenarios more convenient, cortoscript allows for types
to specify a custom search scope for their values. This scope is searched
_before_ `/corto/lang`.

An enumeration could specify itself as the search scope, so that the above
declaration can be rewritten as:

```cpp
Color my_color: Red
```

The exact semantics of the search scope are defined by the type system.

#### Identifier resolution order
This provides a summary of the order in which corto searches in various scopes
for an identifier:

```
type scope -> /corto/lang -> current + parent of current -> /corto
```

The `type scope` is only applicable when assigning a value. If an
identifier is found in a scope, the subsequent scopes are not searched. Scopes
will never be searched twice, thus if the current scope is `/corto/lang`, it
will not be searched again after `/corto/lang` was already searched. Similarly,
if the current scope is a child of `/corto`, `/corto` will not be searched
again after the current- and parent scopes have been searched. Fully qualified
identifier will always be immediately looked up from the root

## Cortoscript and JSON
The following example shows the same data in both cortoscript and JSON:

### Model file
In cortoscript:
```c++
struct Point {
    x, y: int32
}
```

In JSON:
```json
{
    "id": "Point",
    "type": "struct",
    "scope": [
        {"id": "x", "value": {"type": "int32"}},
        {"id": "y", "value": {"type": "int32"}}
    ]
}
```

### Configuration file
In cortoscript:
```c++
driver/mnt/filestore config/fs (
    storedir: "/tmp/data",
    query (
        select: "*",
        from: "data"
    )
)
```

In JSON:
```json
{
    "id": "config/fs",
    "type": "driver/mnt/filestore",
    "value": {
        "storedir": "/tmp/data",
        "query": {
            "select": "*",
            "from": "data"
        }
    }
}
```

## Cortoscript and OMG IDL
The following example shows the same data in both cortoscript and IDL:

IDL:
```
struct Point {
    long x;
    long y;
}
```

cortoscript:
```
struct Point {
    x: int32
    y: int32
}
```

## Code examples
The following section shows various examples of cortoscript.

### Simple model
The following snippet shows how to define a simple struct type:

```c++
struct Point {
    x: int32
    y: int32
}
```
The above snippet declares an object `Point` of type `struct`. It then opens the
scope of the `Point` object, in which two objects `x` and `y` are declared. Both
objects are of type `member`, which is implicit here (`member` is the
`defaultType` of `struct`). Both `x` and `y` are assigned the value `float64`,
which is assigned to the first member of the `member` type, which is `type`.

When this example is fully written out, it looks like this:

```c++
struct Point {
    member x: type:int32
    member y: type:int32
}
```

### Model with inheritance
The following snippet shows how to create a type that inherits from another
type, in this case the `Point` type from the previous example:

```c++
struct Point3D: Point {
    z: int32
}
```

This sample uses the same syntax rules as the previous example to assign a value
to the `Point3D` object. The `Point` object is assigned to the first member of
`struct`, which is `base`. Note how both syntax and member order of type system
objects is carefully chosen to mimic type definitions in other languages.

When this example is written out in full, it looks like:
```c++
struct Point3D: base:Point {
    member z: type:int32
}
```

An alternative notation uses `()` instead of `:`. This notation is better suited
for complex data structures that span multiple lines, but can be used here as
well:
```c++
struct Point3D (Point) {
    z (int32)
}
```

Note that `()` is used to open the *object value*, whereas `{}` is used to open
the *object scope*. The object scope contains child objects that have their own
lifecycles, whereas the object value contains members that are bound to the
lifecycle of the object (and usually occupy the same chunk of memory).

### Simple data instantiation
Using the `Point` type from the previous example, there are various ways in
which the type can be instantiated. The syntax allows for both simple one-liners
as well as for more complex multi-line statements:
```c++
// Simple one-line statement
Point p: 10, 20

// One-line statement with member names
Point p: x:10, y:20

// Multi-line statement with member names
Point p (
    x: 10,
    y: 20
)

// Anywhere where ':' is used to assign a value, '()' can also be used
Point p: x(10), y(20)
```

### Nested data instantiation
The following snippet shows a nested data type and subsequently how it can be
instantiated. It also shows how multiple objects can be assigned the same value
in a single statement.
```c++
struct Line {
    // Assign 'Point' to both 'start' and 'stop' objects
    start, stop: Point
}

// One-liner
Line l: start(x:10, y:20), stop(x:30, y:40)

// Multi-liner
Line l (
    start(x:10, y:20),
    stop(x:30, y:40)
)
```

### Collection types
The following code snippet shows how to create a collection type, and various
ways to instantiate it:
```c++
list IntList: int32

// Using same single-line syntax as used in previous examples
IntList my_list: 10, 20, 30

// Using dedicated syntax for collections
IntList my_list [10, 20, 30]
```

### Anonymous collection types
The following code snippet shows how to use anonymous collection types. Note how
the syntax of an anonymous object is the same as a normal object, but with the
identifier omitted:

```c++
list(int32) my_list [10, 20, 30]
```

When this example is written out in full, it would look like this:

```c++
list(elementType: int32) my_list [10, 20, 30]
```
Note that you cannot use the short-hand `:` notation for opening the value of
anonymous objects, though you can use `:` to set its members.

### Using member modifiers
The corto type system allows for specifying modifiers on members that change the
visibility and access (amongst others) of members. The following code snippet
shows how they can be specified:

```c++
struct Car {
    x: int32, readonly
    y: int32, readonly
}
```

When this example is written out in full, it would look like this:

```c++
struct Car {
    member x: type:float64, modifiers:readonly
    member y: type:float64, modifiers:readonly
}
```
As you can see, `modifiers` is just another member of the `member` type that can
be set using the normal object notation

### More member options
The following snippet shows yet another example of how the object notation can
be used to set more attributes of `member` instances:
```c++
struct Car {
    speed: float64, unit:mph
    turbo: bool, default:"false"
    temperature: float64, unit:celsius, tags[temperature]
}
```
Note that the `tags` member is a collection, and therefore can use `[]` to set
its value.

### Mix combined assignments with object-specific assignments
Cortoscript allows for assigning multiple objects the same value, while also
allowing for object-specific assignments in the same statement:
```c++
struct HQ {
    latitude(default:"37.7749"), longitude(default:"122.4194"): float64
}
```

This capability also comes in handy when manually setting constant values of an
enumeration:
```c++
enum Color {
    Red(0xFF000), Green(0x00FF00), Blue(0x0000FF)
}
```

When object-specific assignments overlap value with the statement-wide
assignment, the object-specific assignment takes precedence. That enables
setting a default value, only when no specific value is set:

```c++
int32 a(10), b(20), c, d: -1 // Only c and d are set to -1
```

### Create a class that implements an interface, adds a method and a constructor
The following example is slightly more complex and shows how to create an
`interface` object with a method that is subsequently implemented by a `class`
object that implements the interface, and adds a constructor.

```c++
interface Movable {
    move(float64 x, float64 y)
}

class Vehicle: implements[Movable] {
    move(float64 x, float64 y)
    construct(): int16
}
```
Notice that `implements` is a member of a collection type, as we
can implement more than one interface.

When we write out this example in full it becomes apparent that methods follow the same syntax as other objects (only repeating the class for brevity):

```c++
class Vehicle: implements[Movable] {
    method move(float64 x, float64 y)
    method construct(): returnType:int16
}
```
The difference with non-procedure objects is that methods have an argument list
which (just like in the object store) is considered part of the object
identifier (to allow for overloading).

### Override methods
The following code snippet shows how to override a method. Overriding a method
is done by creating an `override` object. You can only override methods that
have been created as `overridable` objects.

```c++
class Vehicle {
    overridable move(float64 x, float64 y)
}

class Car: Vehicle {
    override move(float64 x, float64 y)
}
```

### Single-line definitions with a scope
Scopes can be opened and closed on the same line as the declaration. In this
case care must be taken to use a `;` to close the last statement in the scope,
as the corto grammar mandates that each (non-scope) statement must end with
either a semicolon or newline.

```c++
enum Color {Red; Yellow; Green; Blue; Purple;}
```

### Anonymous struct types
The following code snippet shows how to use anonymous struct types. Just like
with anonymous collections, the name is omitted. However, because struct members
are defined in the struct scope instead of the value, we open a scope ('{}'):

```c++
struct{x, y: int32;} p: 10, 20
```
Notice the `;` inside the scope, as we are defining a scope on a single line.

We could define anonymous structs that inherit from anonymous structs:
```c++
struct(struct{x, y: int32;}){z: int32;} p: x:10, y:20, z:30
```

### Hierarchical model
This code snippet demonstrates how cortoscript can be used to describe a
hierarchical datamodel.

```c++
// Drone is a top-level object
container Drone {
    // Tags enable 3rd party plugins to interpret this data
    latitude: float64, tags[latitude]
    longitude: float64, tags[longitude]
    altitude: float64, unit:feet, tags[altitude]

    // Each drone has exactly one battery
    leaf Battery {
        charge_level: uint8, unit:percentage
        temperature: float64, unit:temperature
    }

    // Each drone may have any number of rotors
    table Rotor {
        rpm: uint16
    }
}
```

### Hierarchical objects
This code snippet demonstrates how cortoscript can be used to describe an object
hierarchy.

```c++
// my_drone is the top-level object
Drone my_drone (
    latitude: 37.7749,
    longitude: 122.4194,
    altitude: 250ft)
{
    // drone_battery is a child of my_drone
    Battery battery (
        charge_level: 89%,
        temperature: 59F)

    // Rotor is a child of my_drone
    Rotor {
        // The Rotor instances are childs of Rotor
        Rotor FrontLeft (rpm: 8000)
        Rotor FrontRight (rpm: 8000)
        Rotor BackLeft (rpm: 8000)
        Rotor BackRight (rpm: 8000)
    }
}
```
