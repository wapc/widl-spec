# WebAssembly Interface Definition Language (WIDL) Specification

## What is WIDL?

WIDL is an [interface definition language](https://en.wikipedia.org/wiki/Interface_description_language) for describing [waPC](https://github.com/wapc) modules. WIDL is also used by code generation tools to create binding code for any programming language supported by waPC. waPC and WIDL aim to streamline the bidirectional communication between a host program (custom application or browser) and one or more guest WebAssembly modules. The end result is a true polyglot ecosystem where several WebAssembly modules are dynamically loaded and communicate with each other using a simple programming model.

## Why WIDL?

We built [waPC](https://github.com/wapc) so developers can pass arbitrary bytes to and from WebAssembly similar to request/reply in HTTP. But before we can build real-world applications using waPC, there needs to be a way to convey rich data consisting of maps, arrays, strings, booleans, and various levels of nested data. Currently, passing these high-level data structures agnostically between browsers, host applications and multi-language Wasm modules is not possible without a schema/IDL and serialization format. Existing tools target APIs and micro-service development or are language specific like [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen) and [as-bind](https://github.com/torch2424/as-bind). We determined that a purpose-built IDL was needed to harness the power of WebAssembly.

### Design goals

* Succinct - Clearly described interfaces and high-level data types
* WebAssembly-first - Support all numeric types (i8-64, i8-64, f32, f64)
* Polyglot - Support several widely used programming languages that target Wasm
* Simplicity - Implement a minimal feature set so that it is easier to use and apply to multiple languages
* Extensibility - Allow for developer extensions to satisfy application-specific requirements

### Thoughts on the future

WebAssembly is a rapidly evolving ecosystem. In time, serialization formats are likely to better support polyglot Wasm. Currently, the WIDL code generator leverages [MessagePack](https://msgpack.org/index.html) as the serialization format. It strikes the right balance between performance and ease of use and is lightweight and easy to implement if a language does not already have a de facto MessagePack library. We have seen use cases requiring JSON and Protobuf also. [Interfaces types](https://hacks.mozilla.org/2019/08/webassembly-interface-types/) are also under development. Since the code generation tool was designed to be "pluggable", we see the serialization format eventually being pluggable and WIDL acting as an intermediate representation.

## The WIDL specification

### Namespace

Declared at the top of the WIDL document, the `namespace` is used to identify and refer to elements contained in the WIDL document. It maybe used by the code generator to target filesystem locations of where to write generated files.

```
namespace "customers"
```

Including a version suffix is the recommended way for your application to support multiple versions.

```
namespace "customers.v1"
```

### Scalar Types

Instead of using the `number` type for all numeric values, WIDL inherits WebAssembly's more specific integer and floating point types:

| WIDL Type  | Description                |
| ---------- | -------------------------- |
| `i8`       | A 8-bit signed integer.    |
| `u8`       | A 8-bit unsigned integer.  |
| `i16`      | A 16-bit signed integer.   |
| `u16`      | A 16-bit unsigned integer. |
| `i32`      | A 32-bit signed integer.   |
| `u32`      | A 32-bit unsigned integer. |
| `i64`      | A 64-bit signed integer.   |
| `u64`      | A 64-bit unsigned integer. |
| `f32`      | A 32-bit float.            |
| `f64`      | A 64-bit float.            |
| `bool`     | A boolean.                 |

WIDL also includes the following special types that are decoded into language-specific types:

| WIDL Type  | Description                                                              |
| ---------- | ------------------------------------------------------------------------ |
| `string`   | a UTF-8 encoded string.                                                  |
| `datetime` | a RFC 3339 formatted date / time / timezone.                             |
| `bytes`    | an array of bytes of arbitrary length.                                   |
| `raw`      | a raw encoded value that can be decoded at a later point in the program. |
| `value`    | a free form encoded value that encapsulates any of the above types.      |

**Language support for `datetime`**

Without [WASI](https://wasi.dev/), a WebAssembly module does not have access to the clock or system's timezone. This means the target language may not have a native Date, Time, Timezone, or Timestamp datatypes. In this case, the code generator will treat them as strings.

### Collections

Types can be encapsulated in arrays and maps with enclosing syntax:

| Collection | WIDL Syntax         | Description                                  |
| ---------- | ------------------- | -------------------------------------------- |
| Array      | `[string]`          | A randomly accessible sequence of values.    |
| Map        | `{string : string}` | A mapping of keys to values. Keys must be an integer or string. Values can be any scalar type or object type. |

**Caution about using maps**: In some languages, like Go/TinyGo, the iteration order on maps is considered "undefined". This means that the serialized bytes can be different for the same data. Keep this in mind if you need to compare or create hashes of the serialized data. Using arrays of an object type that contains `key` and `value` might be preferable.

### Optional values

By default, a declared type is required and if unset, contains a zero value (`0`, `""`, empty array or map). To make any type optional (nullable), follow it with a `?`. For example,  `string?` represents an optional string value.

### Interfaces and Roles

Interfaces define the operations available in a waPC module. They are useful for uni-directional scenarios where the host makes calls to the guest module. The syntax resembles interfaces in familiar programming languages.

```
interface {
  add(addend1: i64, addend2: i64): i64
}
```

For bidirectional communication, it is common for incoming and outgoing calls between modules to be different. Furthermore, calls from a guest to the host may not be handled by the host but instead by another guest module. In this case the host is acting as a bridge between modules. So unlike gRPC and other RPC mechanisms, the concept of client and server do not apply in waPC/WIDL. There are only senders and receivers / callers and callees.

Roles are conceptual groups of operations that allow the developer divide communication up into any number of waPC modules. You name the roles according to your application. For example, a distributed calculator could split each mathematical operation up into different roles implemented by different modules.

```
role Adder {
  add(addend1: i64, addend2: i64): i64
}

role Subtractor {
  subtract(minuend: i64, subtrahend: i64): i64
}

role Multiplier {
  multiply(factor1: i64, factor2: i64): i64
}

role Divider {
  divide(dividend: i64, divisor: i64): i64
}
```

The code generation tool is instructed per role to generate either invokers (caller side) or handlers (callee side).

#### Function and unary operations

Operations can follow two possible structures, functions and unary procedures. The operations shown in the examples above are functions. Functions are applicable when passing in a small number of fields and follow a simple, familiar, and easily read format.

All parameters are named. In this example, we passing in first and last names to create a customer and return a `u64` that represents the customer identifier.

```
role CustomerStore {
  createCustomer(firstName: string, lastName: string): u64
}
```

Let's say the number of fields required to create a customer warrants a wrapper object. This is where [unary operations](https://en.wikipedia.org/wiki/Unary_operation) are preferred. In contrast to functions, unary operations accept a single input.

Instead of using parenthesis `(...)` to enclose zero or many parameters, Unaries use curly brackets `{...}` to enclose a single parameter.

```
role CustomerStore {
  createCustomer{customer: Customer}: u64
}
```

Why the different syntax? In the case of functions, multiple parameters are possible so the arguments must be encapsulated in a wrapper object to serialize over waPC. For unary operations, since there is a single parameter no wrapper object is required and the input object can be serialized directly. This syntax is signaling an optimization for the code generation tool.

Now let's look at how `Customer` can be defined using an object type.

### Object types

The most basic component of a WIDL schema are object types, which represent a kind of object you can pass to or return from operations, and what fields it has. Since waPC and WIDL are polyglot, types are defined in a language-agnostic way. This means that complex features like nested structures and inheritance are omitted by design.

In WIDL, we might declare `Customer` like this:

```
type Customer {
  firstName: string
  middleName: string?
  lastName: string
  address1: string
  address2: string?
  city: string
  zipcode: string
  email: string
  phones: [PhoneNumber]
}
```

### Enumeration types

Enumerations (or enums) are a type that is constrained to a finite set of allowed values.

For `Customer`, we might want to allow multiple mobile, home, or work phone numbers. Here's what a `PhoneType` enum definition might look like.

```
type PhoneNumber {
  number: string
  type: PhoneType
}

enum PhoneType {
  mobile = 0 "Mobile"
  home = 1 "Home"
  work = 2 "Work"
}
```

Each enum value denotes its programatic / variable name, the integer value that is serialized, and a display or friendly name for printing. Note that WIDL does not address internationalization so custom code is required to print the value in multiple spoken languages.

### Descriptions

All elements in WIDL can have descriptions which serve as documentation throughout the document. The code generation tools should also preserve descriptions as documentation/comments where appropriate so that you only need to worry about documenting functionality in one place.

Descriptions can be a single line or multiple lines.

Single line

```
"Encapsulates a phone number and its type"
type PhoneNumber {
  "The phone number"
  number: string
  "The phone type"
  type: PhoneType
}
```

Multiple lines

```
"""
Encapsulates a phone number and its type.
The phone number is a single string value and contains
the country code, area code, prefix, and line number.
"""
type PhoneNumber {
  "The phone number"
  number: string
  "The phone type"
  type: PhoneType
}
```

### Annotations

Annotations are ways of attaching additional metadata to WIDL elements. These can be used in the code generation tool to implement custom functionality for your use case. Annotations have a name and zero or many arguments.

Here is what `Customer` might look like with annotations.

```
type Customer {
  firstName: string @notEmpty
  middleName: string?
  lastName: string @notEmpty
  address1: string @notEmpty
  address2: string?
  city: string @length(2)
  zipcode: string @length(5)
  email: string @email @range(min: 5, max: 80)
  phones: [PhoneNumber]
}
```

Multiple annotations can be attached to an element. All annotations have named arguments; however, there are two shorthand syntax options.

| Shorthand    | Equivalent to       | Comments                              |
| ------------ | ------------------- | --------------------------------------|
| `@notEmpty`  | `@notEmpty()`       | Useful for "has annotation" checks.   |
| `@length(5)` | `@length(value: 5)` | `value` is the default argument name. |

The annotation examples above shows a validation scenario but annotations are not limited to this purpose. The developer has the freedom to extend the code generation tool to leverage annotations for their application's needs. In the `Customer` example, the developer could use these annotations to generate `validate` methods on each of the generated object types.

### Default values

Fields can also specify default values when a type is initialized. This needs to be carefully considered in the code generation. Languages vary of how this would be implemented.

```
type PhoneNumber {
  number: string
  type: PhoneType = mobile
}
```

## Why not other IDLs/formats?

Our goal was not to create "yet another IDL". In our WebAssembly journey, we considered several options and ran into issues where the technologies were not perfectly aligned to our design goals:

* Succinct - Clearly described interfaces and high-level data types
* WebAssembly-first - Support all numeric types (i8-64, i8-64, f32, f64)
* Polyglot - Support several widely used programming languages that target Wasm
* Simplicity - Implement a minimal feature set so that it is easier to use and apply to multiple languages
* Extensibility - Allow for developer extensions to satisfy application-specific requirements

<table>
    <thead>
        <tr>
            <th>Specification</th>
            <th>Alignment to goals</th>
            <th>Comments</th>
        </tr>
    </thead>
    <tbody valign="top">
        <tr valign="top">
            <td valign="top">JSON schema</td>
            <td valign="top" nowrap="nowrap">
                <div>❌ Succinct</div>
                <div>❌ WebAssembly-first</div>
                <div>✅ Polyglot</div>
                <div>✅ Simplicity</div>
                <div>✅ Extensibility</div>
            </td>
            <td valign="top">
                <ul>
                    <li>Since JSON schema is written in JSON and not a DSL, it can be verbose.</li>
                    <li>JSON only supports <code>number</code> instead of full set of Wasm numeric types.
                <code>bytes</code> need to be Base 64 encoded strings.</li>
                    <li>Not yet available in all languages. TinyGo has <a href="https://github.com/tinygo-org/tinygo/pull/1741">initial JSON support coming</a>.</li>
                AssemblyScript has <a href="https://www.npmjs.com/package/assemblyscript-json">assemblyscript-json</a>.</li>
                    <li>While extensible, it does not feel natural and many validators will return errors for unknown fields.</li>
                </ul>
            </td>
        </tr>
        <tr valign="top">
            <td valign="top">Protocol Buffers / gRPC</td>
            <td valign="top" nowrap="nowrap">
                <div>✅ Succinct</div>
                <div>✅ WebAssembly-first</div>
                <div>❌ Polyglot</div>
                <div>✅ Simplicity</div>
                <div>✅ Extensibility</div>
            </td>
            <td valign="top">
                <ul>
                    <li><code>.proto</code> files are succinct, supports most of the Wasm numeric fields.</li>
                    <li>Currently, there is a lack of protoc plugin support for TinyGo and AssemblyScript.
                    We believe this support will come in time.</li>
                    <li>It is possible to use <code>.proto</code>s as an IDL but there will still be some subtle issues that cause friction.</li>
                    <li><a href="https://github.com/protocolbuffers/protobuf/releases/tag/v3.15.0" target="_blank">Support for optional fields without wrappers</a> was recently added</li>
                    <li><a href="https://developers.google.com/protocol-buffers/docs/reference/csharp/namespace/google/protobuf/well-known-types">WellKnownTypes</a> like Timestamp and Duration may not translate well to the target language. Timezones may not be available in the Wasm environment.</li>
                    <li>Philosophical opinion: Protobuf is used to generate objects intended for serialization, but are not always ideal
                    to use in your business logic. Developers often let Protobuf objects leak into their business logic.</li>
                    <li>We would like to support Protobuf as a format in the future and this is an important distinction to consider. At the very least, WIDL can be used to produce the Protobuf format that complies with your WebAssembly interface.</li>
                </ul>
            </td>
        </tr>
        <tr valign="top">
            <td valign="top">FlatBuffers / Cap'n Proto</td>
            <td valign="top" nowrap="nowrap">
                <div>✅ Succinct</div>
                <div>✅ WebAssembly-first</div>
                <div>❌ Polyglot</div>
                <div>❌ Simplicity</div>
                <div>✅ Extensibility</div>
            </td>
            <td valign="top">
                <ul>
                    <li><a href="https://github.com/google/flatbuffers/pull/6408">A PR is open for AssemblyScript support in FlatBuffers</a> but still does not support Wasm-enabled languages at this time.</li>
                    <li>Not as simple to interact with as plain data classes/structures. Accessor methods must be used.</li>
                    <li>Lots of features that can make code generation cumbersome.</li>
                    <li>In time could be a good option in addition to MessagePack.</li>
                </ul>
            </td>
        </tr>
    </tbody>
</table>

It's important to note that the IDLs/schemas are usually associated with the serialization format. In some cases the IDL is viable but the format is to cumbersome to implement in all WebAssembly-supported languages or are primarily solving for compaction which is not as much of a concern in WebAssembly. The primary objective in Wasm is to copy data from one module to the host or another module as efficiently as possible.

For a while we used [GraphQL schema](https://graphql.org/learn/schema/) and loved its simplicity. That said, expressing Wasm interfaces still had its friction. Finally, we decided create WIDL as a purpose-built IDL for WebAssembly inspired by the simplicity of GraphQL schema. Here is quick summary of how WIDL differs from GraphQL schema.

* Built-in WebAssembly numeric types (i8-64, i8-64, f32, f64) - no scalars required
* Scalars explicitly alias a known type
* Functions can return `void` instead of returning `Boolean` as a workaround
* Fields are required by default instead of optional and `?` is used after the field name to denote that it is optional
* Support for maps
* Operations are defined in a single interface or roles instead of rigid query and mutation operations
* Removed the concepts that do not apply from GraphQL schema (e.g. Queries vs. Mutations, Field arguments, Variables, Fragments)
