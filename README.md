# Cortoscript
Cortoscript is a declarative language designed to describe data and data models in a coding language and technology independent format. It is designed specifically for the purpose of describing data and data models with rich semantics for (I)IoT systems, with built-in support for hierarchies, object relationships, ontologies and unit annotations.

## Installation
A cortoscript parser is available as a number of corto packages. To install these, first install a minimal bake/corto environment is available with this command:

```
curl https://corto.io/install-dev-src | sh
```

Then, to install the cortoscript parser, run:

```
curl https://corto.io/install-cortoscript-src | sh
```

The installation may take a couple of minutes, as it currently builds the ANTLR4 C++ runtime from scratch.

## Getting started

### Atom syntax highlighting
When your code editor is atom, syntax highlighting is available for cortoscript by installing the "language-cortoscript" package (https://atom.io/packages/language-cortoscript). While in atom, press Ctrl + Shift + P, and then type "Install package". Search "cortoscript", and the package should appear.

### A new cortoscript project
Cortoscript projects consist of a model file (`model.corto`) from which code is generated. To create a new cortoscript project, run the following commands:

```
mkdir my_app
cd my_app
bake init
touch model.corto
```

In `my_app/project.json`, change the `type` attribute from `"executable"` to `"application"`. This will turn the project in a managed project, which automates code generation (for more information on managed projects, see bake docs: https://corto.io/doc/bake.html).

Change the contents of the `model.corto` file to this:

```corto
in my_app

enum Color {
    Red, Yellow, Green, Blue
}

class Car {
    vin: string, key
    color: Color
    lat: float64
    long: float64
}
```

To build the project, and generate code from this model, do:

```
bake
```

You should now have a number of new include files in the `include` directory that contain generated code. The `_type.h` header contains the generated C types. The `_api.h` header contains a generated strongly typed API for creating, updating and assigning instances of the type. For more information on this API, see the corto documentation on "https://www.corto.io".

### Cortoscript examples
The following cortoscript snippets demonstrate some of the language features to get you started.

#### Simple composite type
A composite type in cortoscript is a type that is composed out of one or more other types.

```corto
struct Point {
    x: int32
    y: int32
}
```

#### Instantiate a type
Types can both be created and instantiated in the same script. The `x` and `y` members are written out here in full, to show that objects and types use the same syntax (types are first class citizens in cortoscript).

```corto
struct Point {
    member x = {type: int32}
    member y = {type: int32}
}

Point my_point = {x: 10, y: 20}
```

#### Nested composite type
Composite types may be constructed from other composite types.

```corto
struct Line {
    start: Point
    stop: Point
}

Line my_line = {start.x: 10, start.y: 20, stop: {x: 10, y: 20}}
```

#### Primitives
Cortoscript supports a more extensive number of primitive values than for example JSON.

```corto
enum Color {
    Red, Yellow, Green, Blue
}

struct Car {
    vin: string
    engine_running: bool
    color: Color
}

Car my_car = {vin: "Foo", engine_running: false, color: Red}
```

#### Inheritance
While the exact rules on how inheritance works is outside of the scope of cortoscript, it does provide support for it with the `super` keyword. If there are no member name collisions, usage of `super` is optional.

```corto
struct Point3D: Point {
    z: int32
}

Point3D my_point = {super.x: 10, y: 20, z: 30}
```

#### Units
Object values can be annotated with units.

```corto
Car my_car = {speed: 40mph}
```

#### JSON interoperability
Object values can be written as embedded JSON.

```corto
Point my_point = {"x": 10, "y": 20}
```

#### Object hierarchies
The `{ }` operators (without the `=`) open an object "scope", in which child objects can be created.

```corto
Point my_point = {10, 20} {
    Point child_point = {30, 40}
}
```

#### Implicit typing
A declaration may omit a type, in which case the type from the previous declaration will be inherited.

```corto
Point
p = {10, 20}
q = {30, 40}
r = {40, 50}
```

#### Objects with non-string identifiers
By default objects are identified with a string identifier. The following syntax shows how to use identifiers that are not strings.

```corto
enum State {
    Empty, Cross, Circle
}

struct Tile {
    x: uint8, key
    y: uint8, key
    state: TileState
}

Tile
<0, 0>: Cross
<0, 1>: Circle
<0, 2>: Empty
<1, 0>: Empty
<1, 1>: Cross
<1, 2>: Empty
<2, 0>: Empty
<2, 1>: Circle
<2, 2>: Cross
```

#### Collections
The `[ ]` operators indicate collection values.

```corto
list[int32] integers = [10, 20, 30]
```

#### Nested collections
Collections, just like composite objects, can be nested.

```corto
struct Polygon {
    points: list[Point]
}

Polygon my_poly = {
    points: [
        {0, 0},
        {10, 10},
        {-10, -10}
    ]
}
```

#### References
Classes are reference types. When a struct is embedded, its value becomes part of the parent type. When a reference type is embedded, it becomes a reference to another object.

```corto
class Person {
    mother: Person
    father: Person
}

Person
my_mom = {}
my_dad = {}
me = {mother: my_mom, father: my_dad}
```

#### Procedures
Procedures are callable objects with a return type and parameter list. While these properties are represented as data, just like any other object, cortoscript has a dedicated syntax to describe procedures:

```corto
struct Point {
    x: int32
    y: int32

    add(int32 x, int32 y) void
}
```

#### Overridable methods
Though not strictly a feature of cortoscript, the corto type system provides overridable methods. In cortoscript, overridable methods can be created like this:

```corto
struct Point {
    x: int32
    y: int32
    overridable dot() int32
}

struct Point3D: Point {
    z: int32
    override dot() int32
}
```

#### In/out parameters
Parameters in cortoscript can be in, out and inout.

```corto
enum Result { Success; Failure }
get_measurement(in int8 sensor_id, out float64 sensor_value) Result
```

#### Expressions
Cortoscript has full support for a wide range of expressions and operators. Expressions can be either resolved at parse time, or be stored as reusable components as, for example, a filter.

```corto
float64 PI: 3.1415926
float64 D: 10
float64 R: D / 2
float64 Circumference: 2 * PI * R
float64 Area: R * R * PI
```

## Language Reference

### Introduction
The goal of cortoscript is to provide a higher level data description language for (Industrial) IoT systems than what is currently available (JSON, XML, YAML). Current data description languages lack features commonly found in IoT systems, like semantic awareness, object relationships, unit annotations and strong typing.

The lack of support for these features in existing languages causes different systems to implement these concepts in different ways, which makes exchanging information between systems and thus interoperability unnecessarily hard.

Cortoscript introduces a well defined syntax for these and other features. In addition, cortoscript also provides a framework of guidelines around object identifiers and references, expressions and how higher level features like inheritance should be mapped. Following these guidelines further enhances  interoperability between systems.

Cortoscript is a strict superset of JSON. This makes it more easy to use compliant cortoscript encoded data between environments where JSON is widely used, like web applications.

A unique feature of cortoscript is that its syntax is optimized for describing data models. Being able to describe data models is not unique by itself, as these have been mapped to other data definition languages. However, this often results in hard to read and terse code. The cortoscript syntax has been designed to feel natural for both data and data models, without introducing a separate syntax for both.

### Design goals
The following design goals have guided the design of cortoscript, in order of importance.

**The language MUST be easy to read and write**
Cortoscript MUST be designed as a language that humans can easily read and write. Expected usage of the language is defining data models and configuration files, which are tasks that typically involve humans. To enhance readability, the language SHOULD avoid ambiguity at all times.

**The language MUST promote DRY documents**
A characteristic of languages with a simple design is that they often introduce a lot of redundancy in their documents, XML being a prime example of this. Cortoscript MUST promote DRY ("don't repeat yourself") documents, with a syntax that avoids repetition of redundant information. This makes cortoscript both economic to use as a data container, as well as enhancing its readability.

**The language MUST enable interoperability**
Cortoscript MUST enable interoperability by offering a higher level abstraction for describing data. The language should be aware of common practices in describing data and data models across systems, and provide an intuitive and natural way for using these in documents.

**The language MUST be declarative**
The language should not rely on imperative statements to model or describe data. Imperative statements would imply that a language runtime is required to execute these statements, which would vastly increase the complexity of a parser. Furthermore, imperative statements are harder (if not impossible) to map to a wide range of data-centric technologies, like databases.

**The language MUST be technology independent**
Cortoscript should not include features that make it dependent on a programming language or technology ecosystem. Additionally, cortoscript documents MUST be interpreted consistently such that the side effects of a document are always the same within system boundaries, regardless of underlying operating system or architecture.

**The language MUST be easy to map to other technologies**
Cortoscript MUST be conservative in its features to make it easy to map to a wide variety of environments. While it MUST provide a higher level of abstraction (see interoperability) it SHOULD not rely on exotic features that are not commonplace in existing systems.

**The language syntax SHOULD be as small as possible, but not smaller**
In language design there is a delicate balance between the complexity of a language, and the complexity of the resulting documents. Whereas languages like JSON and XML favor simple syntax in exchange for more complexity and redundancy in documents, cortoscript favors a more complex syntax in exchange for documents that are easier to read and write. With that in mind. cortoscript SHOULD be as small as possible, but MUST do so without impairing readability in documents.

**The language SHOULD not be opinionated**
Cortoscript MUST be a general purpose data definition language and therefore SHOULD not apply arbitrary restrictions on documents that may be sensible in one kind of systems, but not in another. While cortoscript MUST provide a baseline set of rules that enhance interoperability, it should not favor or enforce particular design styles.

**Language documents SHOULD be backward and forward compatible**
Cortoscript should allow systems to evolve gracefully by being flexible in how it accepts data from systems with older or newer data model definitions. This includes allowing for reordering of attributes, adding or removing attributes, and changing the type of attributes.

### The Cortoscript parser
The cortoscript reference implementation consists out of a number of corto packages that address the different responsibilities of the parser. These packages are:

 Package | Description
 --------|-------------
[script-parser](https://github.com/cortoproject/script-parser) | The ANTLR4 lexer/grammar definition and generated ANTLR4 parser |
[script-ast](https://github.com/cortoproject/script-ast) | Generates an AST from script-parser
[script-ast-print](https://github.com/cortoproject/script-ast-print) | Translate AST to a string (used for test cases)
[script-declare](https://github.com/cortoproject/script-declare) | Declare corto objects in corto store from AST
[driver-ext-corto](https://github.com/cortoproject/driver-ext-corto) | Driver that lets corto load files with `.corto` extension

Users can use these packages to rapidly create cortoscript integrations with 3rd party systems. To illustrate, the `script-declare` package is a mere 500 lines of code, and is all that is required for mapping the AST to the corto object store.

The `script-parser` and `script-ast-print` can be used with `corto run` to instantly get visibility into the structure of a cortoscript file. For example, to see the AST of a file called `model.corto`, do:

```demo
corto run corto/script/ast/print model.corto
```

If `model.corto` were to contain this:

```corto
struct Point {
    x, y: int32
}
```

... the output of the above command would be:

```demo
statements:
|   Declaration
|   |   type: Identifier
|   |   |   id: 'struct'
|   |   id: DeclarationIdentifier
|   |   |   ids:
|   |   |   |   Identifier
|   |   |   |   |   id: 'Point'
|   |   scope: Scope
|   |   |   statements:
|   |   |   |   Declaration
|   |   |   |   |   id: DeclarationIdentifier
|   |   |   |   |   |   ids:
|   |   |   |   |   |   |   Identifier
|   |   |   |   |   |   |   |   id: 'x'
|   |   |   |   |   |   |   Identifier
|   |   |   |   |   |   |   |   id: 'y'
|   |   |   |   |   initializer: Initializer
|   |   |   |   |   |   values:
|   |   |   |   |   |   |   InitializerValue
|   |   |   |   |   |   |   |   value: Identifier
|   |   |   |   |   |   |   |   |   id: 'int32'
```

This is an exact representation of how the script is translated to the AST, and what the `script-declare` visitor will see. While this output does gets quite large for larger cortoscript files, it is a useful tool for debugging cortoscript extensions.

Simultaneously, the parser can be invoked directly as well like this:

```demo
corto run corto/script/parser model.corto
```

This will print the ANTLR4 lexer and grammar output to the console, and is an exact representation of what the `script-ast` package will see. For the above command and the same `model.corto` file, the output would look like this:

```demo
[@0,0:0='\n',<62>,1:0]
[@1,1:6='struct',<52>,2:0]
[@2,8:12='Point',<52>,2:7]
[@3,14:14='{',<1>,2:13]
[@4,15:15='\n',<62>,2:14]
[@5,20:20='x',<52>,3:4]
[@6,21:21=',',<3>,3:5]
[@7,23:23='y',<52>,3:7]
[@8,24:24=':',<18>,3:8]
[@9,26:30='int32',<52>,3:10]
[@10,31:31='\n',<62>,3:15]
[@11,32:32='}',<2>,4:0]
[@12,33:33='\n',<62>,4:1]
[@13,34:33='<EOF>',<-1>,5:0]
(program (statements \n (statement (declaration (storage_expression (storage_identifier struct)) (declaration_identifier (storage_identifier Point)) (scope { (statements \n (statement (declaration (declaration_identifier (storage_identifier x) , (storage_identifier y)) (initializer_assignment : (initializer_list (initializer_value (expression (assignment_expression (conditional_expression (logical_or_expression (logical_and_expression (or_expression (xor_expression (and_expression (equality_expression (relational_expression (shift_expression (additive_expression (multiplicative_expression (cast_expression (unary_expression (postfix_expression (storage_expression (storage_identifier int32))))))))))))))))))))))) (eol \n)) }))) (eol \n)) <EOF>)
```

### Keywords
While it may seem from the examples that cortoscript has many keywords (`class`, `enum`, ...) these identifiers are actually not defined in the language, but by the underlying type system. Cortoscript itself has a limited number of keywords, which are:

- in
- out
- inout
- use
- as
- typesystem
- null
- true
- false
- nan
- and
- or
- not

### Operations
A cortoscript parser must support a number of operations to correctly execute cortoscript code. Implementing these operations consistently across different cortoscript backends ensures that cortoscript code can be seamlessly shared between technologies. [This API](https://corto.io/doc/Corto_API_reference.html#Corto_API_Reference_Value_Writer) is an example of how these operations are implemented for the corto backend.

```note
A cortoscript backend may use different names or signatures for these operations to conform with conventions inside the backend. This is ok as long as the implementation is functionally equivalent.
```

Now follows an overview of these operations.

#### INSTANCEOF
Check if a specified object is of a specified type.

##### Signature

```corto
instanceof(object type, object instance) bool
```
The instanceof operation MUST return true if the specified object is an instance of the specified type.

An object is an instance of a type when its value can be interpreted as a value of that type, without requiring any transformations or casts.

#### DECLARE
Create a new object with a specified identifier and type in a specified scope.

```corto
declare(object scope, string identifier, object type) object
```
##### Description
A declare operation lets the backend know of the existence of an object and its type without providing it with a value. This allows a backend to *refer* to the object, but not *interpret* the object value, as the object value at this point is still *undefined*.

When the identifier is not yet known by the backend, the declare operation MUST return a new object with the specified identifier and type. Consecutive operations MUST be able to refer to this object by this identifier.

When an object with the specified identifier already exists, the declare operation MUST return the existing object only if the existing object is an instance of the specified type. If the existing object is not an instance of the specified type, the declare operation MUST fail.

The scope specifies the parent object which will contain the object. A parent object may impose restrictions on objects that are created in its scope. When at least one of these restrictions is violated, the declare operation MUST fail.

The scope MAY be `null`, in which case the object will not be added to a parent object. The resulting object may still act as container for other objects. If scope is set to `null`, the specified identifier will be ignored.

The identifier MAY be `null`. In this scenario, the behavior of the declare operation depends on whether scope is `null` or not. If scope is not null, a new object will be created with a random identifier. If scope is `null`, the identifier is ignored and the behavior is as specified in the previous paragraph.

Objects MAY be provided by other cortoscript documents *or* other unspecified sources. When an object already exists, its value MUST not be overwritten by the cortoscript document.

A cortoscript document that, after fully parsed, results in objects with undefined values is invalid.

#### DEFINE
Signal that the object value is defined.

```corto
define(object instance)
```

##### Description
The define operation signals to the backend that an object value has been set, and may be interpreted by other processes. Before the define operation, the object value is *undefined*.

The define operation MUST trigger a constructor, if the object type specifies one, before signaling that the object may be interpreted. The constructor may execute arbitrary code, including code that verifies the validity of the object value. If the constructor fails, the define operation MUST fail.

The operation MUST run any checks to ensure the validity of the object value before signaling that the object may be interpreted, if the backend requires such checks. If at least one of these checks fail, the define operation MUST fail.

When the define operation fails, the cortoscript parser MUST treat the whole document as invalid, and there MUST be no side effects as result of parsing the document.

#### LOOKUP
Lookup an object with the specified identifier.

```corto
lookup(object scope, string identifier) object
```
##### Description
The lookup operation looks up an object in the backend with the specified identifier. The operation MUST be able to lookup any objects that have been declared before the lookup. When the specified identifier is not found in the backend, the lookup operation MUST fail.

The lookup operation MUST be able to parse scoped identifiers, like `foo/bar`, which indicates an object `bar` that is declared in the scope `foo`. Both the `.` and `/` operators may be used to indicate scoping.

The scope parameter lets the lookup operation search relative to a specified scope. This can reflect the current scope of the cortoscript document. An identifier may be either *relative*, or *fully qualified*. A fully qualified identifier forces the lookup operation to search from the root. Fully qualified identifiers must start with `/`, as in `/foo/bar`.

#### INITIALIZER_CREATE
Create an initializer to set the value for an object.

```corto
initializer_create(object instance) initializer
```
When an object is being initialized in cortoscript, an initializer will be created for the object. The initializer_create operation accepts a declared object, and returns the initializer for that object.

When the object is already defined, the initializer_create operation MUST fail. While a cortoscript document may contain an object that is already defined by the backend, a parser MUST not call this function when the object is already defined.

An initializer is a stateful structure that lets the parser initialize the object value field by field. To keep track of the current field, the initializer must contain a cursor. The cursor position is manipulated by the `INITIALIZER_NEXT`, `INITIALIZER_INDEX` and `INITIALIZER_FIELD` operations.

A value can contain one or more levels of nesting. Objects with complex types that embed other complex types have multiple levels of nesting, for example, a collection with a list of structs. To set fields of an object of a nested type, a value  has to keep track of the current level of nesting. The `INITIALIZER_PUSH` and `INITIALIZER_POP` operations control the level of nesting.

#### INITIALIZER_PUSH
Enter the *value scope* of the current field.

```corto
initializer_push(initializer value, bool as_collection)
```

##### Description
If the cursor points to a field that is of a complex type (composite or collection) the push operation enters the *value scope* of that field, so that the members (when composite) or elements (when collection) can be set.

For example, given a type `Line` with two members `start` and `stop`, both of type `Point` which in turn has two members `x` and `y`, in order to set `start.x`, an application has to push twice: once to enter the value scope of `Line`, and once to enter the value scope of `Point`.

This operation MUST fail when the cursor points to a field that is not a complex type. The operation MAY also fail when the field is of a complex **reference** type. An implementation MAY decide to allow for assignment or implicit creation of objects for reference fields when a value scope is opened for a reference field.

When the operation is called with `as_collection` set to `true`, the operation MUST fail if the type of the current field is not of a collection type.

When the object to be initialized is of a complex reference type, the push operation MUST succeed. Note that the operation thus makes a distinction between the base type of the initializer (provided to `INITIALIZER_CREATE`) and the type of a field, as described in the previous paragraph.

When pushing a field of a nullable collection type, the push MUST create the collection object. Pushing in this case is equivalent to initializing the collection value, so that it is no longer null. An example of this would be initializing the field with an empty linked list structure.

When initializing a complex element in a collection value, the push operation MUST create a new element upon entering the value scope, only when the number of elements in the collection is empty. This only applies to collections that support an `append` operation.

#### INITIALIZER_POP
Leave the *value scope*.

```corto
initializer_pop(initializer value)
```

##### Description
This operation should leave the current value scope. Calls to pop MUST be balanced with `INITIALIZER_PUSH`, or this operation must fail. The operation MUST leave the *value scope* in the exact same state as it was before `INITIALIZER_PUSH`.

#### INITIALIZER_NEXT
Progress the initializer to the next field.

```corto
initializer_next(initializer value)
```

##### Description
This operation progresses the cursor of the initializer to the next field. This function MUST fail when the cursor is at the end of the current value scope.

When this operation is called inside a collection value scope and the current cursor points to the last element in the collection scope, a new element shall be added to the collection. This only applies to collections that support an `append` operation.

This operation is equivalent to calling `initializer_index` and for the cursor argument specify the current cursor + 1.

#### INITIALIZER_INDEX
This operation moves the cursor to a field at the specified index.

```corto
initializer_index(initializer value, int32 index)
```

##### Description
This operation moves the cursor to the field at the specified index. The index MAY either refer to a field in a collection or composite value scope. This operation MUST fail when the index is larger than the number of fields inside the current value scope.

When this operation is called inside a collection value scope and the index equals the total number of elements in the collection scope, a new element shall be added to the collection. This only applies to collections that support an `append` operation.

#### INITIALIZER_FIELD
This operation moves the cursor to the field identified by the field expression.

```corto
initializer_field(initializer value, string field_expression)
```

##### Description
This operation moves the cursor to the field that matches the specified field expression. The field expression MAY either refer to a field in a collection or composite value scope. This operation MUST fail when the field expression does not resolve to a valid field.

Field expressions MAY contain one or more member identifiers and indexes. Members are separated by the dot (`.`) operator, while an index is encapsulated by brackets (`[]`). Field expressions MUST interpret the `super` identifier as referring to a base type, in case the type of the value scope has a base type. When `super` is used inside a value scope that does not have a base class, the operation MUST fail.

The following examples are all valid field expressions:

- speed
- [10]
- super
- position.latitude
- lights[5].color
- super.x

Field expressions MUST be interpreted relative to their value scope. A field expression that identifies field `foo.bar` can be referred to as `bar` when inside the value scope of `foo`.

Field expressions MUST not change the current value scope the initializer is in. Even when a field expression refers to a nested field, the value scope of the initializer after calling this operation must be equal to what it was before calling the operation.

When the field expression refers to a field in the current value scope, it MUST be possible to call the `INITIALIZER_NEXT` operation to progress the next field. Calling `INITIALIZER_NEXT` after using a field expression that resolves to a nested field MUST fail the next operation.

#### INITIALIZER_SET_BOOL
Set the value of the current field to a boolean value.

```corto
initializer_set_bool(initializer value, bool value)
```

##### Description
This operation sets the value of the current field to a boolean value. When the current field is not of a boolean type, the value MUST be casted to the type of the current field. When there is no valid cast from the boolean type to the type of the field, this function MUST fail.

#### INITIALIZER_SET_CHAR
Set the value of the current field to a character value.

```corto
initializer_set_char(initializer value, char value)
```

##### Description
This operation sets the value of the current field to a character value. When the current field is not of a character type, the value MUST be casted to the type of the current field. When there is no valid cast from the character type to the type of the field, this function MUST fail.

#### INITIALIZER_SET_SIGNED_INT
Set the value of the current field to a signed integer value.

```corto
initializer_set_signed_int(initializer value, int64 value)
```

##### Description
This operation sets the value of the current field to a signed integer value. When the current field is not of a signed integer type, the value MUST be casted to the type of the current field. When there is no valid cast from the signed integer type to the type of the field, this function MUST fail.

#### INITIALIZER_SET_UNSIGNED_INT
Set the value of the current field to an unsigned integer value.

```corto
initializer_set_unsigned_int(initializer value, uint64 value)
```

##### Description
This operation sets the value of the current field to an unsigned integer value. When the current field is not of an unsigned integer type, the value MUST be casted to the type of the current field. When there is no valid cast from the unsigned integer type to the type of the field, this function MUST fail.

#### INITIALIZER_SET_FLOATING_POINT
Set the value of the current field to a floating point value.

```corto
initializer_set_floating_point(initializer value, float64 value)
```

##### Description
This operation sets the value of the current field to a floating point value. When the current field is not of a floating point type, the value MUST be casted to the type of the current field. When there is no valid cast from the floating point type to the type of the field, this function MUST fail.

#### INITIALIZER_SET_STRING
Set the value of the current field to a floating point value.

```corto
initializer_set_string(initializer value, string value)
```

##### Description
This operation sets the value of the current field to a string value. When the current field is not of a string type, the value MUST be casted to the type of the current field. When there is no valid cast from the string type to the type of the field, this function MUST fail.

#### INITIALIZER_SET_REFERENCE
Set the value of the current field to a reference value.

```corto
initializer_set_reference(initializer value, object value)
```

#### Operation examples
The following examples show cortoscript declarations and the operations a parser should generate to properly create and initialize the object.

##### Primitive assignment
Example cortoscript:

```corto
int32 my_obj: 10
```
Operations pseudo code:

```corto
object obj = declare(root, "my_obj", int32)
initializer init = initializer_create(obj)
initializer_set_unsigned_int(init, 10)
define(obj)
```

##### Composite assignment
Example cortoscript:

```corto
Point my_obj = {10, 20}
```
Operations pseudo code:

```corto
object obj = declare(root, "my_obj", Point)
initializer init = initializer_create(obj)
initializer_push(init, false)
initializer_set_unsigned_int(init, 10)
initializer_next(init)
initializer_set_unsigned_int(init, 20)
initializer_pop(init)
define(obj)
```

##### Composite assignment with members
Example cortoscript:

```corto
Point my_obj = {x:10, y:20}
```
Operations pseudo code:

```corto
object obj = declare(root, "my_obj", Point)
initializer init = initializer_create(obj)
initializer_push(init, false)
initializer_field(init, "x")
initializer_set_unsigned_int(init, 10)
initializer_next(init) // Redundant, but allows for simpler parser code
initializer_field(init, "y")
initializer_set_unsigned_int(init, 20)
initializer_pop(init)
define(obj)
```

##### Collection assignment
Example cortoscript:

```corto
IntList my_obj = [10, 20]
```
Operations pseudo code:

```corto
object obj = declare(root, "my_obj", IntList)
initializer init = initializer_create(obj)
initializer_push(init, true)
initializer_set_unsigned_int(init, 10)
initializer_next(init)
initializer_set_unsigned_int(init, 20)
initializer_pop(init)
define(obj)
```

##### Collection assignment with composite element type
Example cortoscript:

```corto
PointList my_obj = [{10, 20}, {30, 40}]
```
Operations pseudo code:

```corto
object obj = declare(root, "my_obj", PointList)
initializer init = initializer_create(obj)
initializer_push(init, true)
initializer_push(init, false)
initializer_set_unsigned_int(init, 10)
initializer_next(init)
initializer_set_unsigned_int(init, 20)
initializer_pop(init)
initializer_next(init)
initializer_push(init, false)
initializer_set_unsigned_int(init, 30)
initializer_next(init)
initializer_set_unsigned_int(init, 40)
initializer_pop(init)
initializer_pop(init)
define(obj)
```

##### Description
This operation sets the value of the current field to a reference value. When the current field is not of a reference type, the value MUST be casted to the type of the current field. When there is no valid cast from the reference type to the type of the field, this function MUST fail.

### Documents
Cortoscript documents can be recognized by their `.corto` extension. Documents can be either self contained (containing both data model and data) or rely on other cortoscript documents or packages to provide data models and/or data.

There is no explicit syntax for including one document into another document. The reason behind this is that a cortoscript document is just a front-end for instantiating data into a backend. A "document" is a concept that only lives in the parser, and does not exist anymore once the document has been parsed. To facilitate portability across systems, documents should make no assumptions about the location or availability of other documents on a system or machine.

Cortoscript documents may however rely heavily on data structures provided by other cortoscript documents, packages or other sources of datamodels and/or data. The prerequisite for using these as dependencies however, is that they are accessible through the data backend. This way, a system-wide uniform method is enforced upon documents for referencing other sources of data.

#### Document Header
A document header contains information about where objects in the documents will be inserted in the data backend, and which other parts of the data backend will be referenced. This is done respectively through at most one so called `in` statement, and one or more `use` statements.

#### Document Body
A document body is where the data is defined. In cortoscript, data is described in `declaration` statements. Declarations are the only kind of statement that may occur in the document body. Declarations can be nested, by opening a declaration scope. See the section on declarations for more information.

### In statements
The `in` statement specifies the insertion point in the data backend for the objects in the document. A document may contain at most one `in` statement, and it must be the first statement in the document. This is an example `in` statement:

```corto
in data.sanfrancisco.fleet
```

For this `in` statements, all data will be stored under the `data.sanfrancisco.fleet` "scope". A scope is an abstraction for a level in a hierarchy of objects in the data backend. Object identifiers will be resolved relative to the scope provided in the `in` statement.

An `in` statement is actually a declaration preceded by the `in` keyword. If the scope specified in the `in` statement does not yet exist, the parser may create it. Like declarations (see next section), `in` statements may contain a type:

```corto
in package acme.fleet.model
```

This will find or create a `acme.fleet.model` object in the hierarchy with type `package`. Just like regular declarations, if an existing object is found with that identifier that is not an instance of `package`, the declaration will fail.

Just like declarations, `in` statements may also contain initializers to provide an initial value for the object:

```corto
in package acme.fleet.model = {
    author: "Ford Prefect"
}
```

### Use statements
The `use` statement controls how identifiers are resolved in a document. By default, identifiers are first looked up in the type system scope, which is expected to contains _meta types_, which are types that are used to create types (`struct`, `class`, etc). In corto, this scope is `corto.lang`. After that, an identifier will be looked up in the current scope and its parent scopes. For more details, see "Identifier resolution".

The `use` statement adds more scopes to the search path, so that documents do not always need to use full identifiers when referring to a type or object from another scope.

This is an example `use` statement:

```corto
use acme.fleet.model
```

This statement would let a document refer to the `acme.fleet.model.car` object simply as `car`. The `use` statement also allows a document to create an alias for a scope, like this:

```corto
use acme.fleet.model as fleet
```

With this statement, a document would be able to refer to `acme.fleet.model.car` as `fleet.car`.

These `use` statements are only evaluated after an identifier lookup failed by going through the regular mechanisms. Some systems may however want to use cortoscript documents with custom type system that are more specific for a particular usecase. For these usecases, the `use typesystem` statement is provided:

```corto
use typesystem acme.fleet.model
```

This will replace identifier resolution in `corto.lang` (or whatever the type system scope a system is using), and instead always first lookup identifiers from `acme.fleet.model`.

### Declarations
Declaration statements describe data in cortoscript. Like the name suggests, data is described declaratively instead of imperatively.

This is an example of a simple declaration:

```corto
int32 a: 10
```

This declaration has three elements. The first element is the **type**, which in the example is `int32`. The type describes what the object value will contain, which in this case is a single integer. Types need to be either declared in the document or be available from another source before they can be used.

The second element is an object **identifier**, which in the example is `a`. This identifier uniquely identifies the object. Subsequent declarations can refer to this object by using its identifier. After the document is parsed, other components in the system can also use this identifier to refer to the object.

The third element is the object **value**, which in the example is `10`. Depending on the type of the object, this can be either very simple, like in the example, or very complex. This part of the declaration is also referred to as the "initializer".

Initializers can have a fourth **scope** element. Scopes allow for the creation of child objects. The following snippet shows the previous declaration with a scope:

```corto
int32 a: 10 {
    int32 b: 20
}
```
In this declaration, `b` is a *child* of `a`, and `b` is in the *scope* of `a`.

Declarations are order sensitive. When a declaration statement refers to an object created in another declaration statement, the object has to be declared before it can be referenced.

#### Implicit types
The type element is optional. If the type is omitted, the parser will use the type from the previous declaration. This is illustrated in the following snippet:

```corto
int32 a: 10
b: 20
c: 30
```
This snippet contains three declarations. Objects `b` and `c` are declared with an implicit type. In this case, the parser will select the last used type, which in this case is `int32`. Thus, the previous snippet is equivalent to:

```corto
int32 a: 10
int32 b: 20
int32 c: 30
```
Cortoscript allows for a newline between the type and identifier, such that the above declarations can be rewritten in a more readable way, like this:

```corto
int32
a: 10
b: 20
c: 30
```

Another way a type can be derived is through type system rules. The type system may mandate that in the scope of objects of a certain type, a default type is provided. This is illustrated in the following snippet:

```corto
struct Point {
    x: int32
    y: int32
}
```
Here, a scope is opened for the `Point` object of type `struct`. The `struct` type specifies that the default type in the scope of `struct` instances should be `member`. Therefore, `x` and `y` will be declared with the `member` type. The previous snippet is equivalent to:

```corto
struct Point {
    member x: int32
    member y: int32
}
```

#### Forward Declarations
When a declaration omits an initializer, it is a **forward declaration**. Forward declarations DECLARE an object but not DEFINE it (see Operations). A typical use case for forward declarations is to declaratively create object cycles. The following snippet shows an example of creating a cycle using a forward declaration:

```corto
// Forward declaration of 'Foo'
class Foo

class Bar {
    foo_member: Foo
}

class Foo {
    bar_member: Bar
}
```

Now that we have decomposed the basic elements of a declaration, lets look at them in more detail, working our way backwards from the value, to the initializer to the identifier and finally to the type.

#### Identifier Declarations
As both the declaration type and declaration initializer are optional, a declaration may consist of a single identifier. However, due to ambiguities in the language, a single identifier must be followed up by a `;` to be parsed as a declaration. The following snippet is valid syntax for a declaration:

```corto
Foo;
```
Parsing this code will still fail however, as the parser cannot derive a type. Where this kind of declaration comes in handy however, is inside scopes that specify a default type for its children. For example, in corto, the `enum` type specifies that the default type for its children is `constant`, allowing us to do this:

```corto
enum Color {
    Red,
    Green,
    Blue
}
```
Note that this example uses the `,` to delimit identifiers. This is another way to tell the corto parser to treat the single identifiers as declarations.

The previous snippets is equivalent to:

```corto
enum Color {
    constant Red
    constant Green
    constant Blue
}
```

#### Value Families
Cortoscript supports three different kinds of type "families". Type families are how cortoscript abstracts away from specifics of underlying type systems. An underlying type system may for example support multiple collection types, but their values in cortoscript would look the same. The type families are:

1. Primitive values
2. Composite values
3. Collection values

Primitive values are values that cannot be subdivided into smaller parts. An example is a number, like `10`. You cannot divide the number `10` into smaller parts, like `1` and `0` without the value losing its meaning.

Composite values are composed out of values of other (and occasionally the same) types. The different parts of a composite type are called members, and are identified by a member identifier. Composite types may be composed out of any other type, including composite types.

Collection values are sets of values that all are instances of the same type. Collections can be either bounded or unbounded. Values inside a collection are called elements. Elements can either be identified by a key or an index. A key can be any primitive type. An index must be a positive integer lower than the total element count of a collection value.

#### Initializers
Initializers are how values are described in cortoscript. The following snippet shows a declaration with an example initializer:

```corto
Point p = {x: 10, y: 20}
```

Lets decompose the **initializer**, which starts from the `=` operator. The next character is the `{`, which opens an **initializer scope**, which is closed by a `}`. Inside the initializer scope, we find an **initializer list**, which is populated by **initializer fields** (`x: 10`), which are separated by a `,`.

There are three initializer kinds in cortoscript, the **shorthand initializer**, the **composite initializer** and the **collection initializer**. Here are examples of each:

Shorthand initializer:

```corto
Point p: 10, 20
```

Composite initializer:

```corto
Point p = {x: 10, y: 20}
```

Collection initializer:

```corto
Numbers n = [10, 20, 30]
```

The different kinds of initializers enforce the kind of value they initialize. Composite and collection initializers may only initialize composite and collection values respectively. This enhances readability, as it is unambiguous from the code what kind of value is being initialized. Shorthand initializers may initialize any kind of value and are typically used for simple values.

Each initializer kind accepts an initializer list in its scope. Initializer lists are optional for composite and collection initializers. This is illustrated by this partial grammar example:

```
shorthand_initializer
    : ':' initializer_list
    ;

composite_initializer
    : '{' initializer_list? '}'
    ;

collection_initializer
    : '[' initializer_list? ']'
    ;
```

The next section describes the initializer list in more detail. For the exact semantics on

##### Initializer list
An initializer list is a series of **initializer fields**, separated by a `,` and an optional member identifier. All the examples in this snippet are valid initializer lists:

```corto
10, 20, 30
a: 10, b: 20, c: 30
10, b: 20, 30
```

A member identifier can be either an ordinary identifier (`foo`), a nested identifier (`foo.bar`) or a string (`"foobar"`). Member identifiers may include the special `super` identifier, which can be used to disambiguate between members in a base type, when a type supports inheritance. Here are a few examples of nested, string and super member identifiers:

```corto
foo.hello: 10, foo.world: 20
"hello": 10, "world": 20
super.x: 10, super.y: 20, z: 30
```

When no member identifiers are provided, initializer lists automatically progress through the list of fields in a type in the order in which members are specified in the type. The current field of the initializer list is called the "cursor". The cursor is progressed by the comma operator, or can be set explicitly to a location with a member identifier.

A single initializer list may mix implicit progression (using just the `,` operator) or explicit progression (using member identifiers). A `,` operator after a field with an explicit member identifier will move the cursor to the field after the field identified by the member identifier.

The following three declarations are all equivalent:

```corto
// Use explicit progression (member identifiers)
Point p = {x: 10, y: 20}

// Use implicit progression (just the `,` operator)
Point p = {10, 20}

// Use mixed progression
Point p = {x: 10, 20}
```

When initializing a type with inheritance, the members of the base type come before the members of the sub type. Consider the following type and equivalent declarations:

```corto
struct Point {
    x, y: int32
}

struct Point3D: Point {
    z: int32
}

// Use explicit progression
Point3D p = {super.x: 10, super.y: 20, z: 30}

// Use implicit progression
Point3D p = {10, 20, 30}

// Use mixed progression
Point3D p = {10, super.y:20, 30}
```

When a type contains members that are not writable (read only or private), these are excluded from the initializer list when using implicit progression. Consider the following type and  equivalent declarations:

```corto
struct Car {
    fuel_level: float64, readonly
    latitude: float64
    longitude: float64
    speed: float64
}

Car my_car = {37.7749, 122.4194, 50}
```
Note how in this initializer, `fuel_level` is not part of the initializer as it is a readonly member. Explicitly specifying members that are readonly is invalid. Thus, the following declaration will trigger an error:

```corto
Car my_car = {fuel_level: 40}
```

```warning
Initializers without member identifiers are less robust against changes in data models, so use them only when you can be certain that the type of the declaration is stable. A common use is in type declarations. All of the member declarations use implicit progression (speed: float64) as well as the sub type example (struct Point3D: Point).
```

Collection initializers do not use member identifiers and only use the `,` to progress to the next element. For example:

```corto
Numbers n = [10, 20, 30]
```

When a collection has a dynamic size, progression to the next element is equivalent to appending an element to the collection. An initializer may not contain more elements than the upper bound of the collection type, if the type has one.

Collections may be nullable, in which case `null` is a valid value for the collection. This is shown in the following declaration:

```corto
Numbers n = null
```

Opening a collection scope for a nullable collection is equivalent to creating an empty collection, like is shown in the following snippet:

```corto
Numbers n = []
```

```note
Nullable collections are semantically equivalent to empty collections and may be interpreted as such. The reason for nullable collections is to allow a data backend to skip initialization of collection administration (for example, a linked list object) when it is not going to be used. This increases performance and reduces memory footprint in resource constrained environments. Whether nullable collections are supported or not is up to the data backend.
```

The next section describes the initializer field in more detail.

##### Initializer field
An initializer field can be either an **expression** or a nested initializer. Nested initializers must always be either composite or collection initializers. The following snippets show examples of nested composite and collection initializers:

Nested composite initializer:

```corto
Line l = {start: {10, 20}, stop: {30, 40}}
```

Nested collection initializer with nested composite initializer:

```corto
Poly p = {points: [{x:0, y:0}, {x:10, y:10}, {x:-10, y:10}]}
```

Expressions can be anything ranging from simple literals to more complex binary expressions. Here is an examples of an identifier with both literal and binary expressions:

```corto
Circle c = {radius: 10, surface: 10 * 10 * PI}
```

For more information about possible expressions, see "expressions".

#### Identifiers
Declaration identifiers identify the object being declared. The following snippet shows an example of a declaration with a simple identifier:

```corto
int32 my_obj: 10
```

In this declaration, `my_obj` is the identifier. A single declaration can create multiple objects, by providing multiple identifiers, separated by a `,`. The following declaration declares two objects, `foo` and `bar`, both with value `10`:

```corto
int32 foo, bar: 10
```

##### Parameter Lists
Declaration identifiers may include an **parameter list**. Parameter lists are a specialized syntax that allows for the declaration of "callable" objects, like functions. It is up to the data backend how parameter lists are translated into data. Parameter lists may contain zero or more parameters. The following snippet shows an example of a declaration initializer with an argument list:

```corto
function add(int32 a, int32 b): int32
```

A parameter list may contain `in`, `out` and `inout` parameters by adding the respective keywords to the parameter declarations. By default parameters are `in` parameters. The following snippet shows an example of `in` and `out` parameters:

```corto
// Return false if division by zero, store result in "result"
function div(in int32 num, in int32 by, out int32 result) bool
```

Parameter lists may accept object references. Adding a `&` to the parameter type forces the parameter to be an object, even if the parameter type is of a value type. This snippet shows an example of a reference parameter:

```corto
function add_to_int(int32& obj, int32 value) void
```

##### Non-alphanumeric identifiers
By default, objects are identified by "alphanumeric identifiers", which start with a letter, and can be followed by letters, underscores and numbers. Cortoscript also allows objects to be created with non-alphanumeric identifiers, for example identifiers that just consist out of a number. The following snippet shows an example of two declarations with a numerical identifiers:

```corto
int32 <1>: 10
int32 <2>: 20
```

Non-alphanumeric identifiers may be composed out of multiple elements. The following snippet demonstrates this:

```corto
int32 <"foo", 1>: 10
int32 <"bar", 2>: 20
```

#### Types
A declaration type is a regular identifier that must refer an object specified in the document or known to the backend. Additionally, the object must be a type. In corto, this means the object must be an instance of `corto.lang.type`. A declaration type may be either a named or anonymous.

A type can be either an ordinary type or a meta type. Meta types are a group of types which share the property that their instances are also types. Common examples of meta types are `struct`, `class`, `list`.

The following snippet shows declarations with different kinds of types:

```corto
// Named type
Point p

// Meta type
struct Point

// Anonymous type
list[int32] Numbers
```

#### Scopes
Scopes allow for the creation of child objects. The following snippet shows an example of an object (`a`) with a scope:

```corto
int32 a: 10 {
    int32 b: 20
}
```
In this declaration, `b` is a *child* of `a` and `a` is the *parent* of `b`, and `b` is in the *scope* of `a`. Scopes can be used to build intricate hierarchies of objects, and can at the same time act as a namespace to ensure identifiers do not clash.

Scopes may enact certain policies upon the objects inside the scope. The exact semantics of these policies are determined by the backend. A few examples of such policies could be:

- a default type for objects when no explicit type is provided
- constraints on the type of the child objects
- constraints on the number of child objects
- object cannot contain a scope

When using implicit types inside a scope, a scope does not inherit the implicit type from the parent scope. The following snippet illustrates this:

```corto
int32 a: 10 {
    int32
    b: 30
    c: 40 {
        d: 50 // Error: cortoscript cannot determine type for "d"
    }
}
```
While in this example the parser can infer the type for `c` (`int32`), `d` is the first declaration in a new scope, and thus the parser cannot infer the type for `d`.

### Expressions

#### Literals
The following literals are supported by cortoscript:

| Kind | Example |
|------|---------|
| boolean | `true`, `false` |
| character | `'a'`, `'\0'` |
| integer | `10`, `0`, `-10` |
| floating point | `10.5`, `105e-1` |
| hexadecimal | `0xFF0000` |
| string | `"Hello World"` |
| null | `null` |
| nan | `nan` |
| reference | `foo`, `my_car`, `int32` |

Numeric literals may be annotated with a unit. It is up to the backend how units are interpreted. The following snippet illustrates this:

```corto
Car my_car = {speed: 50mph}
```

#### Operators
Cortoscript supports a range of binary operators. A design goal was to allow a subset of the language to be be used as a generic purpose expression parser, and thus some of the binary operators have no or limited use inside object initializers. The precedence of operators is equal to the C standard.

The following binary operators are supported:

##### Assignment operators
- `=`
- `*=`
- `/=`
- `%=`
- `+=`
- `-=`
- `<<=`
- `>>=`
- `&=`
- `^=`
- `\|=`

##### Arithmetic operators
- `+`
- `-`
- `*`
- `/`
- `%`

##### Bitwise operators
- `&`
- `|`
- `^`

##### Conditional operators
- `&&`, `and`
- `||`, `or`
- `<`
- `>`
- `<=`
- `>=`
- `==`
- `!=`

##### Shift operators
- `<<`
- `>>`

##### Unary operators
- `!`, `not`
- `&`
- `*`
- `+`
- `-`
- `~`
- `!`

##### Postfix operators
- `++`
- `--`
- `.`
- `[]`

##### Ternary operator
- `?` `:`

## Glossary
| Term | Meaning |
|------|---------|
| Object | A uniquely identified unit of data that can be created, updated and deleted. |
| Object value | The state of an object at a given moment in time. |
| Scope | Object container that holds child objects. |
| Initializer | A structure that enables setting the value of an object. |
| Type | An object that describes the syntax and semantics of its instances. |
| Meta type | A type of which the instances are types. |
| Type system | A cohesive collection of meta types. |
| Model | A cohesive collection of types. |
| Primitive value | A non dividable value. |
| Composite value | A value that is composed out of one or more values. |
| Collection value | A value that is composed out of zero or more homogeneous values. |
| Field | A primitive value inside an object. |
| Element | A uniquely identified value inside a collection value. |
| Member | A uniquely identified value inside a composite value. |
