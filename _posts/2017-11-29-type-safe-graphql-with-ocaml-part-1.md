---
layout: post
title: Type-Safe GraphQL with OCaml (part 1)
---

In July 2016, I spent some time writing a GraphQL endpoint for an existing application in Go. I chose to implement the endpoint in Go based on prior experience, good concurrency support and the desire to have static type system. The GraphQL library for Go, [`graphql-go`](https://github.com/graphql-go/graphql), worked pretty well, and I was able to build the desired functionality. Still, I was left frustrated by the lack of type safety offered in Go. Everywhere the library neccessitated the use of `interface{}` types, and the application was littered with type casts. Code like the following was the rule rather than the exception:

```go
func(p graphql.ResolveParams) (interface{}, error) {
  ctx := p.Context.(AppContext)           // Type cast 1
  account := p.Source.(Account)           // Type cast 2
  userId := p.Args["user_id"].(int)       // Type cast 3

  return ctx.LoadUser(account.Id, userId) // Any return value accepted
}
```

Ugh, three type casts to implement a very simple function. Secondly, the return type is `interface{}`, so there's no type safety for the return value. Type casts and lack of type safety collectively mean a much higher risk of runtime errors.

Initially, I thought the lack of generics in Go was the root of the issue. Out of curiosity, I looked into [`graphql-java`](https://github.com/graphql-java/graphql-java), and noticed that it offered equally bad type safety guarantees. I went on to look at [Flow](https://flow.org/) to see if it was possible to get typing for `graphql-js`, but unfortunately it did not (and still does not) support GraphQL ([Github issue](https://github.com/facebook/flow/issues/2823)). This was the starting point of a journey to try to implement a GraphQL library with better type safety guarantees than were offered at the time.

My intent is to write a number of blog posts, which describe that journey. Incidentally, it also describes the implementation of [`ocaml-graphql-server`](https://github.com/andreas/ocaml-graphql-server), which has become the product of the journey. This blog post, part 1, provides a foundation for understanding GraphQL from a server perspective, and describes a type-safe implementation of a subset of GraphQL in OCaml.

## GraphQL Primer

This section provides a primer to a simplified version of GraphQL, which will provide a foundation for later describing how to implement a GraphQL server library.

GraphQL requires you to model your data as a directed graph. As a very simple example, we might have a user with two properties, id and name:

<div class="graphviz-wrapper">
  <img src="/public/graphql-example.svg" />
</div>

To capture the structure of this graph, we need to describe nodes, edges and the relationship between the two more accurately:

- A leaf node is called a *scalar* (circle-shaped). It has a name and a *serialize* function of the type `x -> json`. We call `x` the *source type* of the scalar. The serialize function thus converts from the type `x` to a JSON value, e.g. from `int` to a JSON number.
- An internal node is called an *object* (box-shaped). It has a name, a source type and a collection of *fields*.
- Each field is represented by an edge in the graph. A field has a name and a *resolve function*. The resolve function must be of type `x -> y`, where `x` is the source type of the tail and `y` is the source type of the head. It thus converts from the source type of the head to the source type of the tail.

Nodes are collectively called *types* in GraphQL, i.e. scalars and objects are GraphQL types, and you will also find the terms "scalar type" and "object type" being used.

> **Example**<br/>
> *Int* is a scalar type with source type `int`. *User* is an object type with source type `user`. A field (edge) from *User* to *Int* must have a resolve function of type `user -> int`.

One node in the graph is designated as the query root (named `query` in the above example). The root node must be an object with "nothing" as its source type (i.e. `unit` in OCaml, `void` in C/Java, etc). A query is then a traversal in the graph starting at the root. The result of a query is a JSON value, which can be computed by running the following recursive procedure:

- A scalar node is converted to a primitive JSON value (string, number, boolean, null) by applying its serialize function to the source value.
- An object node is converted to a JSON object with one member per traversed field (outgoing edge). The name of a member is the name of the field, and the value is computed by applying the resolve function to the source value and recursively applying this procedure.

From an inductive perspective, scalars are the base case, while objects require recursive application.

> **Example**<br/>
> Executing a query on the above graph could yield the following JSON value:
>
```json
{
  "user": {
    "id": 1,
    "name": "Alice"
  }
}
```

A final and important aspect of GraphQL is *introspection*. A client can issue a query to a GraphQL API asking for its structure and receive a JSON result describing the graph, i.e. the nodes and edges. We will omit the exact details of this format, but provide the following example.

> **Example**<br/>
> Introspecting the above example graph could yield a JSON value like the following:
> 
```json
{
  "queryType": "Query",
  "types": [
    {
      "name": "Query",
      "kind": "OBJECT",
      "fields": [
        {
          "name": "user",
          "type": "User"
        }
      ]
    },
    {
      "name": "User",
      "kind": "OBJECT",
      "fields": [
        {
          "name": "id",
          "type": "Int"
        },
        {
          "name": "name",
          "type": "String"
        }
      ]
    },
    {
      "name": "Int",
      "kind": "SCALAR"
    },
    {
      "name": "String",
      "kind": "SCALAR"
    }
  ]
}
```

## Implementing the Core of GraphQL

Broadly speaking, the goal of a GraphQL server library is to:

1. Allow the user to construct a schema (graph) in terms of objects, scalars and fields with application-specific serialize and resolve functions.
1. Allow executing a query against a schema.
1. Allow introspection of a schema.

The challenge is to define OCaml types for GraphQL types and fields, which capture the above requirements. Furthermore, the definition needs to support introspection, so we cannot just have a graph of closures, which hide the structure of the graph itself. Finally, the following invariant for fields should be upheld to guarantee that the schema is well-formed:

> **Field Invariant**<br/>
> Given a field with a resolve function of type `x -> y`, the source type of the tail must be `x` and the source type of the head must be `y`.

If we can capture this invariant in the type system, then only well-formed GraphQL schemas will compile and we can avoid exceptions when executing a query.

As a running example, we'll try to expose a simple user type as a GraphQL object:

```ocaml
type user = {
  id   : int;
  name : string;
}
```

We'll be using the following simple definition of a JSON type:

```ocaml
type json =
  | Null
  | Int    of int
  | Float  of float
  | String of string
  | Bool   of bool
  | Array  of json list
  | Object of (string * json) list
```

Given this JSON type, we could implement a simple conversion from `user` to `json` as follows:

```ocaml
(* user_to_json : user -> json *)
let user_to_json user =
  Object [
    ("id",   Int user.id);
    ("name", String user.name);
  ]
```

On to implementing our GraphQL library! A GraphQL type is parameterized over a source type, so we will have a type like `'src typ` (`type` is a reserved keyword, so we use `typ` instead). First we will try to tackle scalars and then add objects. Scalars have a name and a serialize function to convert their source type into JSON:

```ocaml
module Graphql = struct
  type 'src typ =
    | Scalar of {
        name      : string;
        serialize : 'src -> json;
      }
end
```

GraphQL defines a number of built-in scalars, e.g. string and int, which we can define as follows using the above definition:

```ocaml
module Graphql = struct
  type 'src typ = (* ... *)

  (* int Graphql.typ *)
  let int = Scalar {
    name      = "Int";
    serialize = fun i -> Int i;
  }

  (* string Graphql.typ *)
  let string = Scalar {
    name      = "String";
    serialize = fun s -> String s;
  }
end
```

Before expanding our definition of `Graphql.typ` to include objects, let's try to define a field. A field has a name, an output type, and a resolve function:

```ocaml
module Graphql = struct
  type 'src typ = (* ... *)

  and ('src, 'out) field = {
    name        : string;
    output_type : 'out typ;
    resolve     : 'src -> 'out;
  }
end
```

The type `('src, 'out) field` can be interpreted as a field going from a node with source type `'src` to a node with source type `'out`. Note that the return value of `resolve` agrees with the source type of the field `output_type` (our field invariant).

We can then try to define the two fields of our `user`-type:

```ocaml
(* (user, int) Graphql.field *)
let id_field = Graphql.Field {
  name       = "id";
  output_typ = Graphql.int;
  resolve    = fun user -> user.id
}

(* (user, string) Graphql.field *)
let name_field = Graphql.Field {
  name       = "name";
  output_typ = Graphql.string;
  resolve    = fun user -> user.name
}
```

Though these definitions might seem fine, the fact that they have a different types might cause concern for the observant reader. In particular, this means we cannot put them in the same list:

```ocaml
(* TYPE ERROR! *)
let user_fields = [id_field; name_field] 
```

The implication is that we cannot construct a list of fields for an object with the above field types. The insight to overcome this issue is that while we need `'out typ` and `'src -> 'out` to agree, we otherwise do not care about `'out`. In particular, `'out` does not need to be exposed in the type of `field`. We can achieve this using a [GADT](http://caml.inria.fr/pub/docs/manual-ocaml-400/manual021.html#toc85):

```ocaml
module Graphql = struct
  type 'src typ = (* ... *)

  and 'src field = Field : {
    name        : string;
    output_type : 'out typ;
    resolve     : 'src -> 'out;
  } -> 'src field
end
```

By "forgetting" `'out`, the type of `id_field` and `name_field` are now both of type `user Graphql.field`. This means we can create a list containing both user fields:

```ocaml
(* user Graphql.field list *)
let user_fields = [id_field; name_field]
```

With the above in place, we have made it really easy to extend our `Graphql.typ` to include object types. An object is simply a name and a list of fields that agree with the source type of the object:

```ocaml
module Graphql = struct
  type 'src typ =
    | Scalar of {
        name      : string;
        serialize : 'src -> json;
      }
    | Object of {
        name   : string;
        fields : 'src field list;
      }

  and 'src field = Field : {
    name        : string;
    output_type : 'out typ;
    resolve     : 'src -> 'out;
  } -> 'src field
end
```

With these definitions, we can finally construct our GraphQL user type:

```ocaml
(* user Graphql.typ *)
let user_typ = Graphql.Object {
  name = "User";
  fields = [id_field; name_field];
}
```

The final step is being able to convert a GraphQL type to JSON given a value of the corresponding source type. OCaml generally requires no type annotations due to great type inference capabilities. However, the use of GADTs sometimes necessitates use of type annotations. In the following code snippet, you can read `'a. 'a -> ...` as "for all *a*":

```ocaml
module Graphql = struct
  (* ... *)

  let rec to_json : 'src. 'src -> 'src typ -> json =
    fun src typ ->
      match typ with
      | Scalar s ->
          s.serialize src
      | Object o ->
          let members = List.map (resolve_field src) o.fields in
          Object members

  and resolve_field : 'src. 'src -> 'src field -> string * json =
    fun src (Field field) ->
      let field_src  = field.resolve src in
      let field_json = to_json field_src field.output_type in
      (field.name, field_json)
end
```

We've reached the crescendo! The ultimate proof is the ability to convert a user to JSON using `Graphql.to_json`:

```ocaml
let user = { name = "Alice"; id = 1 } in
Graphql.to_json user user_typ
- : json = Object ([("id", Int 1), ("name", String "Alice")])
```

Victory! Though it could seem like we've just replicated the very simple functionality of `user_to_json` in a more complex manner, what we've achieved is much more significant:

- A composable system with type safety.
- A system that can be introspected.
- An extensible system which can support more complex GraphQL types, such as lists, unions, interfaces and non-nullability.

These are all topics for future blog posts.

## Conclusion and Next Up

This blog post describes a composable, extensible core for a GraphQL server implementation, which allows introspection and ensures to runtime exceptions by capturing complex invariants in the type system. In turn, this means users of the library can enjoy compile-time guarantees and build high-quality GraphQL APIs.

Future blog posts will build more features on top of this core, adding support for:

- Schema introspection.
- GraphQL List and NonNull.
- GraphQL Union and Interface.
- Arguments to resolve functions.
- Self-recursion and mutual recursion.
