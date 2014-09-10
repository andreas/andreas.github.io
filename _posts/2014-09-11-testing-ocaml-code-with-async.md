---
layout: post
title: Testing Ocaml code with Async
---

In the [previous blog post](...), we used [Core](https://github.com/janestreet/core) and [Async](https://realworldocaml.org/v1/en/html/concurrent-programming-with-async.html) to write a tiny library for talking to Memcached using the binary protocol. I wanted to write tests for the library in a readable and succint manner to ensure correctness -- the type system cannot ensure binary data is parsed correctly after all. The regular go-to tool for testing Ocaml code is [OUnit](http://ounit.forge.ocamlcore.org/api-ounit/index.html), but this doesn't work well with Async. As I couldn't find a suitable library, I decided to write something myself.

### The Goal

My goal was to write a tiny testing framework with the following features:

- Compatible with Async.
- Assertions with informative output on failure and type-aware equality.
- Before test suite setup function, e.g. connect to Memcached.
- Before test setup function, e.g. prepare some data in Memcached.
- After test teardown function, e.g. flush Memcached.
- After test suite teardown function, e.g. close connection to Memcached.

Inspired by [Command](https://realworldocaml.org/v1/en/html/command-line-parsing.html), I also wanted write the tests in a DSL-like manner. This is an example of the end goal:

{% highlight ocaml %}
Async_suite.(
  create_suite
    ~before_all:(fun () ->
      Memcached.connect "localhost" 11212
    )
    ~before:ident
    ~after:(fun cache ->
      Memcached.flush cache
    )
    ~after_all:(fun cache ->
      Memcached.close cache
    )
  
  |> test "set" (fun cache ->
    Bool.assert_equal (Cache.set cache "foo" "bar") true
  )

  (* add more tests here ...
  |> test "foo" (fun cache ->
    ...
  )
  *)

  |> run
)
{% endhighlight %}

Here we're using [Local Opens](http://caml.inria.fr/pub/docs/manual-ocaml-400/manual021.html#toc77) to avoid prefixing function names with the module name all the time, i.e. `Async_suite.(f x y z)` is the same as `Async_suite.f x y z`. We're also using the pipe function `x |> f`, which is the same as `f x`.

### The Implementation

The type of a test suite is parameterized over the argument for the `before_all` and `after_all` functions, and the argument for the `before` and `after` functions:

{% highlight ocaml %}
(* the type of a single test *)
type 'a test = string * ('a -> unit Deferred.t)

(* test functions should raise AssertionFailure if assertions fail *)
exception AssertionFailure of string

(* the type of a test suite *)
type ('suite_params, 'params) t = {
  before_all : unit -> 'suite_params Deferred.t;
  before     : 'suite_params -> 'params Deferred.t;
  after      : 'params -> unit Deferred.t;
  after_all  : 'suite_params -> unit Deferred.t;
  tests      : 'params test list
}
{% endhighlight %}

Note that a test function is expected to return a `unit Deferred.t`  and should raise `AssertionFailure` if an assertion fails.

Creating the test suite and adding tests is straight forward:

{% highlight ocaml %}
(* Create a empty test suite with before- and after-hooks *)
let create ~before_all
           ~before
           ~after
           ~after_all = {
  before_all;
  before;
  after;
  after_all;
  tests = []
}

(* Add a test function to a test suite *)
let test name f t = { t with tests = (name, f)::t.tests }
{% endhighlight %}

Now we're just missing functionality to run the tests:

{% highlight ocaml %}
(* run_test_with_suite : ('a, 'b) t -> 'a -> 'b test -> unit Deferred.t *)
let test_with_hooks t suite_params (name, test) = 
    t.before suite_params                              >>= fun params ->
    try_with ~extract_exn:true (fun () -> test params) >>= fun result ->
    t.after params                                     >>| fun () ->
    match result with
    | Ok () ->
        printf "%s: PASS\n" name
    | Error (AssertionFailure s) ->
        printf "%s: FAIL\n%s\n" name s
    | Error exn ->
        printf "%s: ERROR\n%s\n" name (Exn.to_string exn)

(* run : ('suite_params, 'params) t -> unit Deferred.t *)
let run t =
  t.before_all () >>= fun suite_params ->
  let tests = List.rev t.tests in
  let run_test = test_with_hooks t suite_params in
  Deferred.List.iter ~how:`Sequential tests ~f:run_test >>= fun () ->
  t.after_all suite_params
{% endhighlight %}

In brief, iterate sequentially over each test function while making sure to call the before- and after-handlers at appropriate times. Test functions are expected to raise `AssertionFailure` if the test fails, which is then caught and printed -- otherwise the test is deemed a pass.

### Writing Assertions

Raising `AssertionFailure` can be done writing plain Ocaml code, but I've found it very helpful to define some assertion-functions myself. The following code makes it easy to assert on values of type `Deferred.Or_error.t` (which is used in the Memcached-client) and get helpful output on failures:

{% highlight ocaml %}
(* signature for assertable modules *)
module type Assertable = sig
  val equal : t -> t -> bool
  val to_string : t -> string
end

(* AssertionError will be raised if an unexpected exception is raised *)
exception AssertionError of string

(* MakeAsserter-functor will return a module with a suitable assert_equals-
   function given an Assertable *)
module MakeAsserter (M : Assertable) = struct
  let assert_equal actual_dfd expected =
    actual_dfd >>= function
    | Error e ->
        raise (AssertionError (Core.Error.to_string_hum e))
    | Ok actual when M.equal actual expected ->
        return ()
    | Ok actual ->
        let err = sprintf "Expected %s was %s" (to_string expected)
                                               (to_string actual)   in
        raise (AssertionFailure err)
end

(* Make asserters for common types *)
module Int    = MakeAsserter(Int)
module Int64  = MakeAsserter(Int64)
module Bool   = MakeAsserter(Bool)
module String = MakeAsserter(String)
{% endhighlight %}

The signature `Assertable` ensures we can compare and print values of some type `t`. This signature is coincidentally fulfilled by all the common data types in Core. The functor `MakeAsserter` allows us to create asserters for any `Assertable` with suitable behavior.

These asserters make it easy to write concise assertions:

{% highlight ocaml %}
String.assert_equal (Cache.get  cache "mykey") "foo"
Int64.assert_equal  (Cache.incr cache "count") 42L
{% endhighlight %}

### Wrap up

In this blog post we wrote a tiny library for testing code that uses Async. The resulting library makes it easy to write concise tests with life-cycle hooks and helpful test output. I wrote this out of a need to test Async code and couldn't find a suitable test library, but if I missed something, please write a comment and enlighten me.
