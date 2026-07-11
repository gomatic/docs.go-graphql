---
title: go-graphql
---

**GraphQL schema tooling for Go: parse SDL into an AST, index and compose schemas, turn introspection responses into SDL, and normalize operations by lifting inline literals into variables.**

- **Source:** [gomatic/go-graphql](https://github.com/gomatic/go-graphql)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-graphql](https://pkg.go.dev/github.com/gomatic/go-graphql)

The module is layered. The root `graphql` package parses SDL text into a [gqlparser](https://github.com/vektah/gqlparser) AST. The `schema` subpackage builds field-lookup indexes, routes root fields across a named set of schemas, and converts introspection JSON to SDL. The `normalize` subpackage rewrites operation text — pulling inline literals out into variables — and revalidates the result. Everything works on in-memory inputs with no shared mutable state; you own all persistence.

## Install

```sh
go get github.com/gomatic/go-graphql
```

## Usage

### Parse SDL

`graphql.Parse` records the source name so parse errors point somewhere useful, and wraps any failure in the `ErrParse` sentinel.

```go
package main

import (
	"errors"
	"fmt"

	graphql "github.com/gomatic/go-graphql"
)

func main() {
	doc, err := graphql.Parse("schema.graphql", "type Query { hello: String }")
	if err != nil {
		if errors.Is(err, graphql.ErrParse) {
			// handle a malformed schema
		}
		return
	}
	fmt.Println(doc.Definitions[0].Name) // Query
}
```

`Parse` takes a `graphql.Name` and `graphql.SDL` (both `string` newtypes) and returns an `*ast.SchemaDocument`.

### Index a schema

`schema.NewIndex` parses SDL and returns an `Index` for read-only field lookup. It parses but does not fully validate, so a partial schema still yields a usable index.

```go
idx, err := schema.NewIndex("api", "type Query { user(id: ID!): User } type User { id: ID name: String }")
if err != nil {
	return
}

idx.HasField("Query", "user")            // true
idx.ArgType("Query", "user", "id")       // ID!
```

`Index` exposes `ArgType`, `HasField`, `GraphQLSchema`, `RootTypeNameForOperation`, and `Schema`.

### Normalize an operation

`normalize.Process` parses, validates, and normalizes a query against an `Index`, lifting inline literals into variables and confirming every field belongs to a single schema.

```go
res, err := normalize.Process(idx, `{ user(id: "42") { id name } }`)
if err != nil {
	return
}

res.Query()         // the rewritten operation
res.Variables()     // literals lifted into variables
res.HasVars()       // whether any were extracted
```

Use `normalize.ProcessWithSchemaHint` when routing against a composite index and you already know which member schema to prefer.

## Design

- **Layered, single-responsibility packages.** SDL parsing (`graphql`), schema indexing and composition (`schema`), and operation normalization (`normalize`) are separate; each builds only on the layers beneath it.
- **`Composite` routing.** `schema.NewComposite` takes an ordered set of named schemas and routes a query's root fields to their owning schema, so a normalized operation is guaranteed to target one schema.
- **Introspection to SDL.** `schema.IntrospectionToSDL` and `schema.IndexFromIntrospection` turn a GraphQL introspection response into SDL or a ready-to-use index.
- **Sentinel errors.** Failures are matchable with `errors.Is` against exported constants such as `graphql.ErrParse`.
