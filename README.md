# cortoscript specification
Cortoscript is the language that is used by the corto framework to describe
models and data that instantiates models. Typical usecases for cortoscript are:
- data model definitions
- API definitions
- configuration data
- expression parsing

The purpose of cortoscript is to provide a concise and readable method for
natively describing corto objects and models. Parsers that do the same exist
for notation languages like JSON and XML, however definitions in these formats
are comparatively verbose and hard to read.

There are several scenarios where cortoscript can add value. Firstly, many
corto projects define their own datamodel and API specification. Cortoscript
natively understands the corto typesystem, which means it is the most concise
method of describing model files.

A second scenarios is when cortoscript is used to specify configuration data. In
this case, objects are described in a script that when instantiated run some
logic in their constructors that activates a particular component. A typical
usecase for this is to instantiate mounts or other infrastructure objects.

Thirdly, cortoscript can be used to parse expressions, which can then be ran by
applications dynamically. A typical usecase for this is to write a filter in
cortoscript, and parse and evaluate the filter at runtime. In this scenario,
cortoscript is combined with a backend that can dynamically execute code, like the corto virtual machine (https://github.com/cortoproject/vm).

This repository contains a cortoscript parser with a formal definition of the
grammar: https://github.com/cortoproject/script-parser

## Design principles
Cortoscript uses the corto API underneath to instantiate objects. Therefore the
same rules apply that apply to declaring/defining objects in the object store
also apply to cortoscript. Every statement in cortoscript has an equivalent API
function (or set of API functions) associated with it.

Cortoscript assumes that the underlying type system is self-describing, and thus
does not define a separate DSL for defining types. Types in cortoscript use the
same syntax that is used to instantiate object. This decouples cortoscript from
the type system, and allows cortoscript to evolve independently from the corto
runtime.

Cortoscript aims to look and feel familiar. Even though the language does not
have a separate DSL for defining types, the syntax has been chosen in such a way
that when a type is written down using the normal object notation, it still
resembles type definitions from other popular languages, like C++ or Java.

Cortoscript leverages type information to keep syntax concise. It does not, for
example, mandate that field names are provided when initializing objects, as is
the case with for example JSON. That also means that cortoscript cannot be
parsed without an accompanying type definitions. Those can be either provided
in the same file, or come from another file or corto package.

## Language overview
This section does not aim to provide an exhaustive overview of the language, as
the language grammar is a more appropriate and concise way to do that. Here we
describe the high-level design of the language, so that a reader will be able
to read and write definitions in cortoscript.

### Declarations
Cortoscript is a declarative language, which means that scripts use declarations
to instantiate data. Declarative languages declare the "things that should be",
whereas an imperative language the "things that should be done". Here is a
simple C example that demonstrates the differences between declarative and
imperative styles:

Imperative:

```c
Point p;
p.x = 10;
p.y = 20;
```

Declarative:

```c
Point p = {.x = 10, .y = 20};
```

In cortoscript, a declaration takes one of the following forms (simplified):

```c++
type identifier: value
type identifier (value)
type (value)
```

The notation that uses `:` is referred to as the "shorthand" notation, and
allows object values to be specified on a single line. The notation that uses
`()` is called the "composite notation", and is better suited for complex
declarations that span multiple lines.

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
    start(x:10, y:20),
    stop(x:30, y:40)
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

In addition to the composite notation, there is also a "collection" notation
which, like the name suggests, allows for specifying collection values. The collection notation uses uses `[]` instead of `()` to encapsulate a value. The
following example demonstrates an object that contains a collection value:

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
    (start(x:10, y:20), stop(x:30, y:40)),
    (start(x:50, y:60), stop(x:70, y:80))
]
```

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

### Defining types
A unique property of cortoscript is that it does not have a separate DSL for
specifying types, as mentioned in the design goals. To get an idea for what
this means in practice, consider the following type description:

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

There are various rules in the type system that determine whether an object can
be assigned to a reference of a specific type. The aforementioned `object` type
accepts references of any type, whereas an instance of a `class` type only
accepts instances of that class. This is more exhaustively described in type
system documentation.

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
adopted in the industrial space.

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
        Person father: mother:grandmother
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
Person my_family/my_household/father: mother:my_family/grandmother
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
type scope -> /corto/lang -> current + parent scopes -> /corto
```

The `type scope` is only applicable when assigning a value. If an
identifier is found in a scope, the subsequent scopes are not searched. Scopes
will never be searched twice, thus if the current scope is `/corto/lang`, it
will not be searched again after `/corto/lang` was already searched. Similarly,
if the current scope is a child of `/corto`, `/corto` will not be searched
again after the current- and parent scopes have been searched. Fully qualified
identifier will always be immediately looked up from the root

## Cortoscript and JSON
In some scenarios, both cortoscript and JSON can be used, like for example when
creating configuration files or model files. In many situations, cortoscript
will provide a more concise, easier to read alternative to using JSON as it can
leverage the full semantics of the corto type system.

The ubiquity of JSON, in combination with its self-describing nature however
means there are also plenty of scenarios where JSON is a better choice. As a
general guideline, cortoscript is preferred when the file is exclusively used
within a corto context, for example, a file that defines the datamodel for a
corto project.

When the data needs to be shared across projects outside of the corto ecosystem,
JSON is a better choice. An example is data that is sent to a web client
or server.

The following two examples show an example model definition and configuration
in both cortoscript and JSON, for illustration:

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
OMG IDL is a description language for describing structured data, and is widely
used in CORBA (Common Object Request Broker Architecture) and DDS (Data
Distribution Service) based systems. IDL is superficially similar to cortoscript
in that it offers a language-independent way of describing types.

This simple example demonstrates some similarities between cortoscript and IDL:

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

IDL and cortoscript have different design goals. IDL is designed as a language
to describe syntax of a connectivity framework, whereas cortoscript is designed
to describe application layer datatypes and data, which are often hierarchical
and incorporate a notion of semantics. These high-level differences result in a
number of technical differences.

The most significant difference is that while IDL allows for describing types,
it cannot be used to describe data. In cortoscript, since there is no meaningful
distinction between types and data, you can describe both datamodels and
the objects that instantiate them.

Secondly, whereas cortoscript adapts itself to the underlying typesystem, IDL is
more rigid. You can look at an IDL grammar definition and see that it has
structs, collection types, and so on. In cortoscript, these constructs are
defined in the type system. The cortoscript language contains no built-in
notion of which kinds of constructs are supported.

Newer version of IDL support annotations, which allows for adding new features
to the language without changing the grammar. These annotations however still
annotate existing constructs. Cortoscript allows introducing new constructs by
extending the type system, which is something that can be done without
changing the corto runtime.

Thirdly, cortoscript is capable of annotating types and values with their
respective units. This enables model and datadefinitions to be written in a way
that guarantees unit consistency when a value is assigned.

Cortoscript is designed to work well with languages like IDL, and in many cases
can be considered a superset of IDL. Where IDL describes syntax, cortoscript
describes syntax, semantics and data. A connectivity layer that uses IDL, like
OMG-DDS, can therefore be naturally extended with semantics using cortoscript.

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
