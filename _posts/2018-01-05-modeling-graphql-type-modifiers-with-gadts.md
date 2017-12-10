---
layout: post
title: Modeling GraphQL Type Modifiers with GADTs (part 2)
---

This blog post is the second in a series, which describes how to implement a type-safe GraphQL library in OCaml (see [part 1 here](/posts/2017-29-11-type-safe-graphql-with-ocaml)).

In part 1, we defined two OCaml types, `Graphql.typ` and `Graphql.field`, to represent GraphQL objects, scalars and fields. We identified a field invariant which guarantees at compile-time that a schema is well-constructed: resolve functions must return values of the right type. In this part we will extend `Graphql.typ` to handle non-nullable types and list types -- collectively known as *type modifiers*.

## Recap

Recall our example graph from part 1, a user with an id and a name:

<div class="graphviz-wrapper">
  <img src="/public/graphql-example.svg" />
</div>

To represent this in GraphQL, we came up with the following OCaml type definitions:

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

We also defined a JSON type, which we will be using again in this part:

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

## Null or not null?

In GraphQL, all types are nullable by default. If you declare a GraphQL field to be of type `String`, then `null` is a valid value for that field.

> **Example**<br/>
> When executing the following query against our example schema:
>
```
query {
  user {
    name
  }
}
```
> ... this should be a valid response:
>
```json
{
  "data": {
    "user": {
      "name": null
    }
  }
}
```

In OCaml, there is no concept of `null`. Rather, if a value is optional, an option type is used instead:

```ocaml
type 'a option =
  | Some of 'a
  | None

(* string_option_to_json : string option -> json *)
let string_option_to_json maybe_str =
  match maybe_str with
  | Some str -> String str
  | None -> Null
```

Recall that the type `'src Graphql.typ` means that the source type for that GraphQL type is `'src`, e.g. the value `Graphql.string` has the type `string Graphql.typ` and requires a `string` value.

Our current type definitions do not use option types, so we do not allow any nulls at all! To do so, we need a type such as `'src option Graphql.typ`, which accept either `None` or `Some x`, where `x` is of type `'src`. To implement this, we need to turn to GADTs again.

> **GADT Primer**<br/>
> When reading a GADT such as this:
>
```ocaml
type _ foo =
  | Option : 'a -> 'a option foo
```
> You can consider `Option` a function from type `'a` to type `'a option foo`. Once you've constructed a value of type `'a option foo`, you can match against it and get back a value of type `'a`.
>
> Example:
> 
```ocaml
(* You provide an int so the type of foo_int is int option foo *)
let foo_int = Option 42
>
(* extract : 'a option foo -> 'a *)
let extract foo =
  match foo with
  | Option x -> x
>
extract foo_int
- : int = 42
```
> There's more to GADTs, but this basic explanation will suffice for understanding this blog post.

Previously the definition of `Graphql.typ` was a regular variant type. Now we'll make it into a GADT to allow nulls by default:

```ocaml
module Graphql = struct
  type _ typ =
    | Scalar : {
        name      : string;
        serialize : 'src -> json;
      } -> 'src option typ
    | Object : {
        name   : string;
        fields : 'src field list;
      } -> 'src option typ
end
```

With this definition, the value `Graphql.string` has type `string option Graphql.typ`, and `Graphql.int` has type `int option Graphql.typ`. This means we can now define `name` as a nullable field:

```ocaml
(* user Graphql.field *)
let name_field = Graphql.Field {
  name    = "name";
  typ     = Graphql.string;            (* <-- string option Graphql.typ *)
  resolve = fun user -> Some user.name (* <-- None would be allowed     *)
}
```

Let's assume we want the user id field to *not* allow null though. We can add a third constructor `NonNull` to `Graphql.typ`, which given an nullable type, returns a non-nullable type:

```ocaml
module Graphql = struct
  type 'src typ =
    | Scalar  : (* ... *) -> 'src option typ
    | Object  : (* ... *) -> 'src option typ
    | NonNull : 'src option typ -> 'src typ
end
```

With this in hand, we can now enforce `id` to be non-nullable:

```ocaml
let id_field = Field {
  name = "id";
  output_type = NonNull Graphql.int; (* <-- int Graphql.typ         *)
  resolve = fun user -> user.id      (* <-- cannot return None here *)
}
```

Perfect! The final piece to this part of the puzzle is updating our definition of `Graphql.to_json`, such that it handles the new variant `NonNull`:

```ocaml
module Graphql = struct
  (* ... *)

  (* unless_null : 'a option -> ('a -> json) -> json *)
  let unless_null src f =
    match src with
    | None -> Null
    | Some src' -> f src'

  let rec to_json : 'src. 'src -> 'src typ -> json =
    fun src typ ->
      match typ with
      | Scalar s ->
          unless_null src s.serialize
      | Object o ->
          unless_null src (fun src' ->
            let members = List.map (resolve_field src') o.fields in
            Object members
          )
      | NonNull t ->
          to_json (Some src) t

  and resolve_field : 'src. 'src -> 'src field -> string * json =
    fun src (Field field) ->
      let field_src  = field.resolve src in
      let field_json = to_json field_src field.output_type in
      (field.name, field_json)
```

At this point the type system prevents the user from returning null (`None`) unless the output type of the field is actually nullable. This is an amazingly powerful feature, which both prevents null pointer errors, but also means the library does not have to do null-checks at runtime &mdash; something you'll find in (almost) all other GraphQL server libraries.

## Lists

Consider adding a field `tags` to our GraphQL user object, which should be a list of strings. The output of the field should thus have type `string list Graphql.typ`, which would allow an OCaml value such as `["tag_1"; "tag_2"]`.

We can approach lists very similarly to how we handled nulls, simply by extending our GADT with another constructor:

```ocaml
module Graphql = struct
  type 'src typ =
    | Scalar  : (* ... *) -> 'src option typ
    | Object  : (* ... *) -> 'src option typ
    | NonNull : 'src option typ -> 'src typ
    | List    : 'src typ -> 'src list option typ
end
```

The constructor `List` accepts a GraphQL type and returns a list of that type. As an example, `List Graphql.string` represents a list of strings and has the type `string option list option Graphql.typ`. You might wonder why the type is `string option list option` and not just `string option list`. The reason is that in GraphQL, lists can also be null. This leaves us with four ways to combine list and non-null:

| Type | Example values | Description |
|------|----------------|-------------|
| `List Graphql.string` | `None`<br/>`Some [None]`<br/>`Some [Some "tag_1"; Some "tag_2"]` | Nullable list of nullable strings |
| `List (NonNull Graphql.string)` | `None`<br/>`Some ["tag_1"; "tag_2"]` | Nullable list of non-nullable strings |
| `NonNull (List Graphql.string)` | `[None]`<br/>`[Some "tag_1"; Some "tag_2"]` | Non-nullable list of nullable strings |
| `NonNull (List (NonNull Graphql.string))` | `["tag_1"; "tag_2"]` | Non-nullable list of non-nullable strings |

Lastly we need to update our `to_json`-function to handle the `List`-constructor:

```ocaml
module Graphql = struct
  (* ... *)

  let rec to_json : 'src. 'src -> 'src typ -> json =
    fun src typ ->
      match typ with
      | Scalar s  -> (* ... *)
      | Object o  -> (* ... *)
      | NonNull t -> (* ... *)
      | List t ->
          unless null src (fun src' ->
            let elements = List.map (fun element -> to_json element t) src' in
            List elements
          )
end
```

This is straightforward and the types work out perfectly.

## Conclusion and Next Up

In this post we've extended the GraphQL core from [part 1](/posts/2017-29-11-type-safe-graphql-with-ocaml) to include the GraphQL type modifiers *non-null* and *list*. This is a feature that is typically implemented in GraphQL libraries using reflection or type casting, which in turn results in runtime errors. In OCaml we can model these type modifiers as GADTs, and by doing so, we've been able to maintain the compile-time guarantee that only well-constructed schemas are allowed by the type checker. 

In the next blog post we'll show how to allow introspection of the GraphQL types we've defined so far.
