# WebAssembly Interface Definition Language (WIDL) for Golang

WIDL is a schema format for describing [waPC](https://github.com/wapc) modules and used by the [CLI](https://github.com/wapc/cli) to generate code for the supported guest languages. It heavily resembles [GraphQL schema](https://graphql.org/learn/schema/) but with some variations to fit better in the WebAssembly ecosystem.

* Built-in WebAssembly numeric types (i8-64, i8-64, f32, f64) - no scalars required
* Scalars explicitly alias a known type
* Functions can return `void` instead of returning `Boolean` as a workaround
* Fields are required by default instead of optional and `?` is used after the field name to denote that it is optional
* Support for maps
* Operations are defined in a single interface instead of separating query and mutation operations
* Removed the concepts that do not apply from GraphQL schema (e.g. Queries vs. Mutations, Field arguments, Variables, Fragments)

Everything in this package was borrowed and retrofitted from the awesome [Golang GraphQL library](https://github.com/graphql-go/graphql).  We thank the 70+ contributors to this project! It has enabled us to provide a succinct interface definition language to our users with minimal effort.
