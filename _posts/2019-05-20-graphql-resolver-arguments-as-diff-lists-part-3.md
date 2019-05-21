---
layout: post
title: GraphQL Resolver Arguments as Diff Lists (part 3)
redirect_from: /2019/05/17/graphql-resolver-arguments-as-diff-lists-part-3/
---

This blog post is the third in a series, which describes how to implement a type-safe GraphQL server library in OCaml. In [part 1]({% post_url 2017-11-29-type-safe-graphql-with-ocaml-part-1 %}), we described how define GraphQL scalars and objects in a type-safe manner, and in [part 2]({% post_url 2018-01-05-modeling-graphql-type-modifiers-with-gadts-part-2 %}) we extended this to include the notion of type modifiers (non-null and list).

In practical terms, we could express a schema such as the following at the end of part 2 (using the [GraphQL Schema Definition Language](https://graphql.org/learn/schema/#type-language)):

```
type User {
  id: Int!
  name: String
}

type Query {
  users: [User!]!
}
```

The definition of objects was restricted to only allow fields without arguments, however. That is, defining a field such as `search(query: String!, limit!): [User]` to search for users was impossible.

In this post, we will extend our library such that fields can have arguments, while keeping the guarantee that only well-formed GraphQL schemas will compile.

## Understanding the problem

Before we start adding arguments to fields, let's revisit our type definitions from the previous posts:

```ocaml
module Graphql = struct
  type 'src typ =
    | Scalar : {
        name      : string;
        serialize : 'src -> json;
      } -> 'src option typ
    | Object : {
        name   : string;
        fields : 'src field list;
      } -> 'src option typ
    | NonNull : 'src option typ -> 'src typ
    | List    : 'src typ -> 'src list option typ

  and 'src field = Field : {
    name        : string;
    output_type : 'out typ;
    resolve     : 'src -> 'out;
  } -> 'src field
end
```

Recall that the resolve function of a field transforms the source value to the output value. This is currently captured by the type `'src -> 'out`, which we want to extend to include arguments.

We will also be reusing our JSON type definition from the previous posts:

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

If you consider a field such as `search(query: String, limit: Int): [User]`, there are two arguments:

- An argument named `query` of type `String`.
- Another argument named `limit` of type `Int`.

When submitting a GraphQL query containing a field with arguments, the client must provide a JSON-like value for each argument, e.g.:

```
query {
  search(query: "Alice", limit: 10) {
    id
    name
  }
}
```

Reusing the JSON type definition from the previous parts, here `query` has the value `String "Alice"`, and `limit` has the value `Int 10`.

## Loosely Typed Approach

As a motivating example, let's start with a very simple, loosely typed approach. Arguments are essentially a map from arguments names to argument values. A simple solution would be to simply expose this mapping to the resolve function like so:

```ocaml
module StringMap = Map.Make(String)

module Graphql = struct
  module Arg = struct
    type t = {
      name : string;
      typ  : string;
    }
  end

  type 'src typ = (* ... *)

  and 'src field =
    | Field : {
        name    : string;
        out     : 'out typ;
        args    : Arg.t list;
        resolve : 'src -> json StringMap.t -> 'out;
      } -> 'src field
end
```

Here we've added a new type `Arg.t` to represent arguments, added the field `args` to the `field`type, and further changed the type of `resolve` to `'src -> json StringMap.t -> 'out`.

With this in hand, we can define our `search` field:


```ocaml
(* unit Graphql.field *)
let search_field = Graphql.Field {
  name = "search";
  out = Graphql.List user_type;
  args = [
    { name = "query"; typ = "String" };
    { name = "limit"; typ = "Int" }
  ];
  resolve = fun () args ->
    (* coerce arguments and search ... *)
}
```

The downside of this approach is that the resolve function will have to look up arguments in the map `args`, and coerce values from `json` to more accurate types such as `string` and `int`. Such code is tedious and error-prone to write, and increases the risk of runtime exceptions.

In a sense, we've already provided information about the type of each argument, but unfortunately not in a format that the type checker can pick up. The challenge is now to find a solution, where the type checker can leverage this information so the user is lifted of this burden.

*(As an aside: While this solution is a strawman and may seem naïve, it is not far from the type safety and ergonomics found in many GraphQL libraries)*

## Defining Argument Types

In this section we'll pursue a solution with more fine-grained types, which better captures the connection between thee argument list and the resolve function of a field.

We'll start out with defining a new type for _argument types_. Argument types in GraphQL have similar richness as the rest of the type system, but for the sake of simplicity, this blog post will be limited to scalar argument types without type modifiers. The approach presented here is open for extensibility along the same lines as shown in [part 2]({% post_url 2018-01-05-modeling-graphql-type-modifiers-with-gadts-part-2 %}) though.

The new type `'a Graphql.Arg.typ` is parameterized over the type that will be exposed to the resolve function, e.g. `string` for the GraphQL argument type `String`, or `int` for the GraphQL argument type `Int`:

```ocaml
module Graphql = struct
  module Arg = struct
    type 'a typ = {
      name : string;
      deserialize : json -> ('a, string) result
    }
  end
end
```

The record field `deserialize` converts the client-provided argument value to an internal value of type `'a`, or provides an error message.

We can now define values to represent the GraphQL argument types `String` and `Int`:

```ocaml
module Graphql = struct
  module Arg = struct
    type 'a typ = (* ... *)

    (* string typ *)
    let string = {
      name = "String";
      deserialize = function
        | String s -> Ok s
        | _ -> Error "Invalid input value for scalar String"
    }

    (* int typ *)
    let int = {
      name = "Int";
      deserialize = function
        | Int n -> Ok n
        | _ -> Error "Invalid input value for scalar Int"
    }
  end
end
```

This representation also allows for custom scalar types such as `Date`, `DateTime` or similar.

## Defining Arguments

Now that we have argument types, we can actually define arguments. An argument simply consists of a name and an argument type:

```ocaml
module Graphql = struct
  module Arg = struct
    type 'a typ = (* ... *)

    type 'a t = {
      name : string;
      typ : 'a typ
    }
  end
end
```

At this point, we can express the arguments `query` and `limit` for the field `search(query: String, limit: Int): [User]` in the following manner:

```ocaml
(* string Graphql.Arg.t *)
let query_arg = { name = "query"; typ = Graphql.Arg.string }

(* int Graphql.Arg.t *)
let limit_arg = { name = "limit"; typ = Graphql.Arg.int }
```

Note that `query_arg` and `limit_arg` have different types, so we cannot use the same solution as the "loosely typed approach" of representing arguments as a simple list. We'll address this challenge next.

## Defining Argument Lists

A single field can have many arguments, each of which exposes a value to the resolve function. The next challenge is how to combine multiple arguments while meaningfully capturing their effect on the resolve function.

Intuitively adding an argument of type `'a Graphql.Arg.t` to the field should add an argument to the resolve function of type `'a`. In concrete terms, a field with an argument of type `string Graphql.Arg.t` should have a resolve function of type `'src -> string -> 'out`.

The concept of _diff lists_ comes in handy here. Put briefly, a diff list allows you to create a heterogeneous list (a list containing elements of differing types) by capturing the type of the elements in a type parameter. I'll only treat the subject lightly here, but I recommend [Drup's great blog post](https://drup.github.io/2016/08/02/difflists/) if you want to dig deeper.

Our heterogeneous argument list is expressed using a GADT in the following manner:

```ocaml
module Graphql = struct
  module Arg = struct
    type 'a typ = (* ... *)
    type 'a t   = (* ... *)

    type ('out, 'f) arg_list =
      | []   : ('out, 'out) arg_list
      | (::) : 'a t * ('out, 'f) arg_list -> ('out, 'a -> 'f) arg_list
  end
end
```

The first type parameter `'out` is the output type of operations over the list, while the second type parameter is an arrow type (`'a -> ... -> 'z`), which captures the contents of the list. Rebinding `[]` and `::` allows us to conveniently write `Graphql.Arg.[arg1; arg2; arg2]` ([introduced](https://github.com/ocaml/ocaml/pull/234) in OCaml 4.03).

Here are two small examples:

```ocaml
(* ('a, string -> 'a) Graphql.Arg.arg_list *)
let _ = Graphql.Arg.[query_arg]

(* ('a, string -> int -> 'a) Graphql.Arg.arg_list *)
let _ = Graphql.Arg.[query_arg; limit_arg]
```

This shows how adding elements to the argument list alters the second type parameter to fit the type of the contents. Note the type variable `'a` which is the type of the output when applying an operation over the list, which we'll look at next.

The following function, `apply`, applies a function to an argument list given a mapping from argument names to JSON values:

```ocaml
module Graphql = struct
  module Arg = struct
    (* 'a t -> json StringMap.t -> ('a, string) result *)
    let get t args =
      match StringMap.find_opt t.name args with
      | None -> Error "Missing argument"
      | Some json_value -> t.typ.deserialize json_value

    let rec apply : type out f.
        (out, f) arg_list -> f -> json StringMap.t -> out =
      fun arg_list f args ->
        match arg_list with
        | [] -> Ok f
        | arg :: arg_list' ->
          match get arg args with
          | Error _ as err -> err
          | Ok x ->
              let f' = f x in
              apply arg_list' f' args
  end
end
```

There's a lot going on here, so let's unfold it bit by bit.

The helper function `get` extracts an argument value from the map of JSON values, and deserializes it to a value of type `'a` given an argument of type `'a Graphql.Arg.t`. If the argument is not present in the map, or deserializing the value fails, the whole operation fails (`Error ...`).

The first thing to note about `apply` is the use of [locally abstract types](https://caml.inria.fr/pub/docs/manual-ocaml/extn.html#s%3Agadts), `type out f. ...`, which is required for polymorphic recursion using GADTs. The basic structure of the function is otherwise a simple recursion over the argument list. Keep in mind that this is not a regular list though, but an `(out, f) Graphql.Arg.arg_list` as we've rebound `[]` and `::`.

We'll consider each case separately:

- **Base case** (`[]`): In this case, `arg_list` must be of type `(out, out) arg_list`, which further means `f` must be of type `out`. The computation is done and we can return this value.
- **Inductive case** (`arg :: arg_list'`): Here `arg` is of type `'a t` and `arg_list'` is of type `(out, 'a -> 'g) arg_list`. We further know `f` is of type `'a -> 'g`. By calling `get` with `arg`, we can get a value of type `'a`, and apply it to `f` to get a value `f'` of type `'g`. This means we can recursively call `apply` with `arg_list'` and `f'`.

The most important point is that the type checker ensures that the function `f` agree with the argument list `arg_list`.

Here's a quick example of how this could be used in practice:

```ocaml
let args =
  StringMap.empty
  |> StringMap.add "query" (String "Alice")
  |> StringMap.add "limit" (Int 5)
in
Graphql.Arg.apply
  Graphql.Arg.[query_arg; limit_arg]
  (fun query limit -> Format.asprintf "query = %s, limit = %d" query limit)
  args
```

... which would evaluate to `"query = Alice, limit = 5"`.

Note how the type of the argument list `(string, string -> int -> string) Graphql.Arg.arg_list` matches the type of the applied function `string -> int -> string`. This is incredibly powerful for enforcing that the user-provided resolve function has the right type.

## Putting it all together

We're finally at the point where our argument list can be integrated in the definition of fields:

```ocaml
module Graphql = struct
  module Arg = struct
    type ('out, 'f) Arg.arg_list = (* ... *)
  end

  type 'src typ = (* ... *)

  and 'src field = Field : {
    name        : string;
    output_type : 'out typ;
    arguments   : ('out, 'f) Arg.arg_list
    resolve     : 'src -> 'f;
  } -> 'src field
end
```

The type of `field` ensures that the resolve function accepts exactly the right arguments and returns a result which matches the output type.

The culmination of all our work is the ability to define the `search` field with nice and exact types:

```ocaml
let search_field =
  let open Graphql in
  {
    name = "search";
    output_type = List user_type;
    arguments = Arg.[
      query_arg;
      limit_arg
    ];
    resolve = fun () query limit ->
      (* ... *)
  }
```

Here `query` has type `string` and `limit` has type `int`, just as we would expect from the argument definition. Success!

## It's a wrap

In this post, we've shown how diff lists can be used to capture the connection between the arguments and the resolve function of a GraphQL field. This enables typing fields in a very fine-grained manner to the benefit of type-safety and ergonomics &mdash; no type casts in sight!

The approach presented in this post is generally applicable when you have untyped data, a typed schema, and you want to extract the data in a typed way. You can find numerous other applications of this, such as arguments to command line applications ([cmdliner](https://github.com/dbuenzli/cmdliner)), matches for regular expressions ([tyre](https://github.com/drup/tyre)), or patterns for rewriting the OCaml AST (`Ast_pattern` from [ppxlib](https://github.com/ocaml-ppx/ppxlib)).

Using diff lists does come with a trade-off though. Firstly, the signature of `Graphql` has less educational value for new users due to the complexity of the types &mdash; the interface must be accompanied by usage examples. Secondly, diff lists have some [non-obvious limitations](http://drup.github.io/2016/08/02/difflists/#because-we-can-never-have-nice-things). Thirdly, other features which interact with the resolve function are made more complicated, e.g. adding support for middleware.

In the next blog post we'll show how our representation of GraphQL types and fields allows introspection.
