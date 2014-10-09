---
layout: post
title: Efficient Use of Core Reader and Writer
---

In a [previous blog post](/2014/09/17/improved-ocaml-memcached-client-with-core-and-async/), I used [Core](https://realworldocaml.org/v1/en/html/concurrent-programming-with-async.html), [Async](https://realworldocaml.org/v1/en/html/concurrent-programming-with-async.html) and [bitstring](code.google.com/p/bitstring/) to write a tiny library for talking to Memcached using the binary protocol. [Reader](https://ocaml.janestreet.com/ocaml-core/latest/doc/async/#Std.Reader) and [Writer](https://ocaml.janestreet.com/ocaml-core/latest/doc/async/#Std.Writer) are the primary I/O abstractions used in [Core](https://ocaml.janestreet.com/ocaml-core/latest/doc/) and also underlies my implementation. To better understand the mechanics of Reader and Writer and how to use them efficiently, I decided to examine the implementations and APIs more closely. The following is a summary of my findings.

### Reader

[Reader](https://ocaml.janestreet.com/ocaml-core/latest/doc/async/#Std.Reader) is an abstraction for asynchronously reading data from a file descriptor. It does so by issuing [read system calls](http://linux.die.net/man/2/read), either non-blocking in the same thread or blocking in another thread. Data from the file descriptor is read into an internal buffer of type [`Bigstring.t`](https://ocaml.janestreet.com/ocaml-core/latest/doc/core/#Bigstring).

Here's a simple example for reading into a string buffer:

{% highlight ocaml %}
let buffer = String.create 10 in
Reader.really_read reader buffer >>= function
| `Eof -> printf "Failed\n"
| `Ok  -> printf "Read: %s\n" buffer
{% endhighlight %}

In more details, this is what happens:

1. A string of 10 bytes is allocated.
2. 10 or more bytes is read from the file descriptor into the internal buffer of `reader` using read system call.
3. 10 bytes are blitted from internal buffer of `reader` to `buffer` using [memcpy](http://linux.die.net/man/3/memcpy).

Copying data from the internal buffer to `buffer` may be wasteful if `buffer` is not later mutated. If `buffer` is used in an immutable fashion, we can achieve the same without the wasteful copying:

{% highlight ocaml %}
let handle_chunk buffer ~pos ~len =
  if len >= 10 then
    `Stop_consumed (Bigsubstring.create buffer ~pos 10, 10)
  else
    `Continue
in
Reader.read_one_chunk_at_a_time reader ~handle_chunk >>= function 
| `Eof    -> printf "Failed\n"
| `Ok buf -> printf "Read: %s\n" (Bigsubstring.to_string buf) 
{% endhighlight %}

Using this approach we skip step 1) and 3), as the result is simply exposed as a substring of the internal buffer. No additional copying is done.

Accessing the internal buffer of reader as Bigstring also allows us to read binary data in an easy fashion:

{% highlight ocaml %}
(* handle_chunk : Bigstring.t -> pos:int -> len:int ->                *)
(*                  [`Continue | `Stop of { magic : int; ... } * int] *)
let handle_chunk buffer ~pos ~len =
  if len >= 24 then
    let open Bigstring in
    (* example parsing Memcached header *)
    let header = {
      magic       = unsafe_get_int8     buffer 0;
      opcode      = unsafe_get_int8     buffer 1;
      key_length  = unsafe_get_int16_be buffer 2;
      (* more fields here *)
    } in
    `Stop_consumed (header, 10)
  else
    `Continue
{% endhighlight %}

`Bigstring` has a family of functions for parsing binary values with the naming scheme `unsafe_get_[type]_[endian]`, e.g. `unsafe_get_uint32_be`. Using any function prefixed with `unsafe_` should make you think twice. The documentation has the following note:

> The "unsafe_" prefix indicates that these functions do no bounds checking. [...] In practice, message parsers can check the size of an outer message once, and use the unsafe accessors for individual fields, so many bounds checks can end up being redundant as well.

As we check the buffer length up front we should be safe.

The above approach parses and copies data from the internal buffer, as it as such not zero copy. To achieve zero copying, you could use a library like [cstruct](https://github.com/mirage/ocaml-cstruct), which might be a topic for a future blog post.

### Writer

[Writer](https://ocaml.janestreet.com/ocaml-core/latest/doc/async/#Std.Writer) is an abstraction for asynchronously writing data to a file descriptor. A `Writer.t` maintains a queue of [`Bigstring.t`](https://ocaml.janestreet.com/ocaml-core/latest/doc/core/#Bigstring) to be written to the file descriptor, and issues [writev system calls](http://linux.die.net/man/2/writev) every so often (either based on time or once per scheduler cycle). Data can be added to the queue either by providing Bigstrings directly, or by writing to the internal buffer of the `Writer.t` (type [`Bigstring.t`](https://ocaml.janestreet.com/ocaml-core/latest/doc/core/#Bigstring)).

This is the simplest way to write a string to a Writer is as follows:

{% highlight ocaml %}
Writer.write writer "123"
{% endhighlight %}

Under the covers, this is what happens:

1. The string `"123"` is blitted to the internal buffer of `writer`, which gets added to the writer's queue.
2. Writer will asynchronously write to the file descriptor using writev system call.

If you already have a `Bigstring.t`, it can be added directly to the queue:

{% highlight ocaml %}
let buffer = Bigstring.of_string "123" in
Writer.schedule_bigstring writer buffer
{% endhighlight %}

This avoids all copying, since `buffer` is simply added to the writer's internal queue -- we can thus avoid all copying. Note that since writing to the file descriptor is done asynchronously, it's not safe to modify `buffer` in the meantime.

Like with Reader, the internal buffer of a Writer can be exposed. This is done through the `Writer.write_gen` function:

{% highlight ocaml %}
Bigstring.write_gen : length:('a -> int) ->
                      blit_to_bigstring:('a, Bigstring.t) Blit.blit ->
                      ?pos:int -> ?len:int -> Writer.t -> 'a -> unit

type ('a, 'b) Blit.blit = src:'a -> src_pos:int -> len:int ->
                            dst:'b -> dst_pos:int -> unit
{% endhighlight %}

That is, given a function `length : 'a -> int` and a function `blit_to_bigstring : src:'a -> src_pos:int -> len:int -> dst:Bigstring.t -> dst_post:int -> unit`, `write_gen` will return a function to write a value of type `'a` to the internal buffer of a writer: `?pos:int -> ?len:int -> Writer.t -> 'a -> unit`.

Like for Reader we can use the exposed Bigstring for writing binary data using the family of functions `Bigstring.unsafe_set_[type]_[endian]`, e.g. `Bigstring.unsafe_set_uint32_be`. Let's look at an example for writing a Memcached header:

{% highlight ocaml %}
module Header = struct
  type t = { magic : int; opcode : int; key_length: int (* and more fields *) }

  (* length : t -> int *)
  let length t = 24

  (* blit_to_bigstring : t -> Bigstring.t -> pos:int -> len:int -> unit *)
  (* We'll ignore the len argument for simplicity                       *)
  let blit_to_bigstring t buffer ~pos ~len =
    Bigstring.(
      unsafe_set_int8     (pos+0) t.magic;
      unsafe_set_int8     (pos+1) t.opcode;
      unsafe_set_int16_be (pos+2) t.key_length;
      (* ...and so on *)
    )

  (* write : t -> Writer.t -> unit *)
  let write t writer = Writer.write_gen ~length ~blit_to_bigstring t writer
end
{% endhighlight %}

The same considerations above about safety apply here.

So in terms of efficiency if your data is represented as a Bigstring, it's most efficient to schedule it for writing with `Writer.schedule_bigstring` (or any other `Writer.schedule_*` function). This avoids all copying. If not, then you can either serialize to a Bigstring and schedule it, or write it to the writers internal buffer with `write_gen`. This is not zero copy, but is better than serializing to a string and then writing that to the writer.

### Wrap up

The most efficient use of Reader and Writer is achieved by not copying data needlessly. For Reader this is done with `Reader.read_one_chunk_at_a_time` and returning "views" of the internal buffer. For Writer it's most efficient to schedule Bigstrings for writing with `Writer.schedule_*` or use `Writer.write_gen` to write directly to the writer's internal buffer.

Achieving minimal or zero copying by obeying the mentioned guidelines make APIs a little more cumbersome though. Instead of reading and writing data with strings, reads need to return Bigsubstrings and writes must be done with Bigstrings or Bigsubstrings. Depending on your application it may or may not be worth this extra complexity.

If you like this post, please vote on [Hacker News](https://news.ycombinator.com/item?id=8420071).
