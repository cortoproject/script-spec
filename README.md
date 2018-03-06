# cortoscript specification
Cortoscript is the language that is used by the corto framework to describe
models and data that instantiates models. Typical usecases for cortoscript are:
- data model definitions
- API definitions
- configuration data
- expression parsing

When a cortoscript is parsed, it populates the corto object store with objects
and types. Typically when parsing a data model or API definition it is because
a language-specific representation is generated for those definitions. In that
case, cortoscript is used as a front-end for the code generator.

A second usecase is when cortoscript is used to specify configuration data. In
this case, objects are described in a script that when instantiated run some
logic in their constructors that activates a particular component. A typical
usecase for this is to instantiate mounts or other infrastructure objects.

Lastly, cortoscript can be used to parse expressions, which can then be ran by
applications to dynamically. A typical usecase for this is to write a filer in
cortoscript, and parse and evaluate the filter at runtime. In this scenario, a
cortoscript is parsed to a backend that can dynamically execute code, like the
corto virtual machine (https://github.com/cortoproject/vm).

This repository contains a cortoscript parser with a formal definition of the
grammar: https://github.com/cortoproject/script-parser

## Design principles
Cortoscript uses the corto API underneath to instantiate objects. Therefore the
same rules apply that apply to declaring/defining objects in the object store,
also apply to cortoscript. In other words, every statement in cortoscript has
an equivalent API function (or set of API functions) associated with it.

Cortoscript assumes that the underlying typesystem is self-describing, and thus
does not define a separate DSL for defining types. Types in cortoscript use the
same syntax that is used to instantiate object. This decouples cortoscript from
the typesystem, and allows cortoscript to evolve independently from the corto
runtime.

Cortoscript aims to look and feel familiar. Even though the language does not
have a separate DSL for defining types, the syntax has been chosen in such a way
that when a type is written down using the normal object notation, it still
resembles type definition from other popular languages, like C++ or Java.

Cortoscript leverages type information to keep syntax concise. It does not for
example mandate that field names are provided when initializing objects, as is
the case with for example JSON. That also means that cortoscript cannot be
parsed without an accompanying type definitions, which can be either provided
in the same file, or come from another file or corto package.

## Differences between cortoscript and JSON
In some scenarios, both cortoscript and JSON can be used, like for example when
creating configuration files or model files. In many situations, cortoscript
will provide a more concise, easier to read alternative to using JSON as it can
leverage the full semantics of the corto type system.

The ubiquity of JSON, in combination with its self-describing nature however
means there are also plenty of scenarios where JSON is a better choice. As a
general guideline, cortoscript is preferred when the file is exclusively used
within a corto context. An example is the file that defines the datamodel for a
project.

When the data needs to be shared across projects outside of the corto ecosystem,
JSON is a better choice. Also, needless to say, for any communication with web
applications JSON is clearly the better option.

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
int32 a(10), b(20), c, d: -1
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
Notice that `implements` is a member of a collection type, which means that we
can implement more than one interface.

When we write out this example in full it becomes apparent that methods, while
seeming different, actually follow the same syntax as other objects (only
repeating the class for brevity):

```c++
class Vehicle: implements[Movable] {
    method move(float64 x, float64 y)
    method construct(): returnType:int16
}
```
The difference with previous objects is that methods have an argument list which
just like in the object store is considered part of the object identifier (to
allow for overloading).

### Override methods
The following code snippet shows how to override a method. Overriding a method
is done by creating an `override` object. You can only override methods that
have been created as `overridable` objects.

```c++
class Vehicle {
    overridable move(float64 x, float64 y)
}

class Car {
    override move(float64 x, float64 y)
}
```

### Single-line definitions with a scope
Scopes can be opened and closed on the same line as the declaration. In this
case care must be taken to use a `;` to close the last statement in the scope,
as the corto grammar mandates that each (non-scope) statement must end with a
semicolon or newline.

```c++
enum Color {Red; Yellow; Green; Blue; Purple;}
```

### Anonymous struct types
The following code snippet shows how to use anonymous struct types. Just like
with anonymous collections, the name is omitted, however because struct members
are defined in the struct scope instead of the value, we open a scope:

```c++
struct{x, y: int32;} p: 10, 20
```
Notice the `;` inside the scope, as we are defining a scope on a single line.

We could define anonymous structs that inherit from anonymous structs:
```c++
struct(struct{x, y: int32;}){z: int32;} p: x:10, y:20, z:30
```

### Child objects
This code snippet demonstrates how cortoscript can be used to describe an object
hierarchy.

```c++
Drone my_drone (
    latitude: 37.7749,
    longitude: 122.4194,
    altitude: 250ft)
{
    Battery drone_battery (
        charge_level: 89%,
        temperature: 59F)

    Rotors {
        Rotor FrontLeft (rpm: 8000)
        Rotor FrontRight (rpm: 8000)
        Rotor BackLeft (rpm: 8000)
        Rotor BackRight (rpm: 8000)
    }
}
```
