---
layout: post
title: Improved Memcached client with Core and Async
private: true
---

In the [previous blog post](/2014/08/22/implementing-the-binary-memcached-protocol-with-ocaml-and-bitstring/), we implemented a simple Ocaml library for talking to Memcached with the binary protocol using [bitstring](https://code.google.com/p/bitstring/). The code uses the baked-in standard library and synchronous IO (blocking), so a lot of time will be wasted waiting for IO. The standard library replacement [Core](https://github.com/janestreet/core) offers cooperative threading with callbacks through [Async](https://github.com/janestreet/async), similar to Javascript, [EventMachine](http://rubyeventmachine.com/) for Ruby or many others. In this blog post we'll try to rewrite the code from the previous post to use asynchronous IO with [Async](https://github.com/janestreet/async).

### IO with Async

The primary IO abstractions in the Ocaml standard library are [in\_channel and out\_channel](http://caml.inria.fr/pub/docs/manual-ocaml/libref/Pervasives.html#6_Inputoutput), while in Core/Async it's [Reader](https://ocaml.janestreet.com/ocaml-core/111.17.00/doc/async/#Std.Reader) and [Writer](https://ocaml.janestreet.com/ocaml-core/111.17.00/doc/async/#Std.Writer). Reading and writing with Core/Async is asynchronous (non-blocking), so results are not returned immediately. If we for example examine the signature of `Reader.really_read` it looks like this:

{% highlight ocaml %}
(* really_read t pos len buffer reads until it fills len bytes of buffer starting at
   pos or runs out of input.` *)
val really_read : Reader.t -> ?pos:int -> ?len:int -> string ->
                    [ `Eof of int | `Ok ] Deferred.t
{% endhighlight %}

The type `'a Deferred.t` signifies that the result of type `'a` is not immediate, but will be filled in sometime in the future. To actually access the result, you can bind functions to be called when the Deferred is resolved using `Deferred.bind`:

{% highlight ocaml %}
(* Deferred.bind : 'a Deferred.t -> ('a -> 'b Deferred.t) -> 'b Deferred.t *)

let buf = String.create 10 in
let dfd = Reader.really_read my_reader 0 10 buf in
Deferred.bind dfd (function
  | `Ok    -> return buf
  | `Eof _ -> return ""
)
{% endhighlight %}

Note that `bind` applies a function to the resolved value and must return a new Deferred value. That's why we're using `return buf` (type `string Deferred.t`) instead of just `buf` (type `string`). The type of `return` is `'a -> 'a Deferred.t`.

To make things more readable, there are two handy infix operators for working with Deferreds:

{% highlight ocaml %}
(* the infix operator >>= is an alias of bind                      *)
(* >>= : 'a Deferred.t -> ('a -> 'b Deferred.t) -> 'b Deferred.t   *)
dfd >>= function
  | `Ok    -> return buf
  | `Eof _ -> return ""

(* the infix operator >>| is a shortcut to avoid explicit return   *)
(* >>| : 'a Deferred.t -> ('a -> 'b) -> 'b Deferred.t              *)
dfd >>| function
  | `Ok    -> buf
  | `Eof _ -> ""
{% endhighlight %}

Let's turn to `Writer`. The most basic function it offers is `write`:

{% highlight ocaml %}
(* write : Writer.t -> ?pos:int -> ?len:int -> string -> unit *)
Writer.write my_writer "foo"
{% endhighlight %}

You might notice that `write` does not return a deferred: `write` only queues the write in a buffer, and another Async microthread actually writes to the OS buffer. If you want to ensure that the write has been flushed, you can call `Writer.flush : unit -> unit Deferred.t`. An exception will be raised if any errors occur while transferring data to the OS buffer. How to handle exceptions in Async is a topic of another blog post (if you're curious [start here](https://ocaml.janestreet.com/ocaml-core/latest/doc/async/#Std.Monitor)).

We've covered enough to update the client library to use Async now, but if you want to learn more about Async, you can also read [Dummy's Guide to Async](http://janestreet.github.io/guide-async.html) from Jane Street or the excellent chapter on Async in [Real World Ocaml](https://realworldocaml.org/v1/en/html/concurrent-programming-with-async.html).

### Updating the Client

We can reuse the code from [the last post](), which doesn't touch IO. All the functions `read_*` or `write_*` needs to be rewritten to be asynchronous though. Let's start by writing a packet:

{% highlight ocaml %}
(* write_packet : Writer.t -> packet -> unit *)
let write_packet writer packet =
  let header_bits = header_to_bitstring packet.header in
  Writer.write writer (Bitstring.to_string header_bits);
  Writer.write writer (Bitstring.to_string extras);
  Writer.write writer packet.key;
  Writer.write writer packet.value
{% endhighlight%}

This is almost identical to the previous version and doesn't even introduce any deferreds. Note that we have to convert our bitstrings to strings though, while we could previously emit them directly to the channel (e.g. `Bitstring.to_chan out_chan header_bits`). The bitstring library was not built with Async in mind.

Reading the response bears less resemblance with the former version and we now have to use deferreds:

{% highlight ocaml %}
(* read_header : Reader.t -> header option Deferred.t *)
let read_header reader =
  let header_buffer = String.create 24 in
  Reader.really_read reader ~len:24 header_buffer >>| function
    | `Eof _ -> None
    | `Ok    -> Some (header_of_bitstring (Bitstring.of_string header_buffer)) 

(* read_body : header -> Reader.t -> packet option Deferred.t *)
let read_body header reader =
  let body_length  = Int32.to_int header.body_length                        in
  let value_length = body_length - header.extras_length - header.key_length in
  let body_buffer  = String.create body_length                              in
  Reader.really_read reader ~len:body_length body_buffer >>| function
    | `Eof _ -> None
    | `Ok    -> 
      bitmatch Bitstring.of_string body_buffer with
        | { extras : 8*header.extras_length : bitstring;
            key    : 8*header.key_length    : string;
            value  : 8*value_length         : string
          } -> Some { header; extras; key; value; }
        | { _ } -> None

(* read_response_packet : Reader.t -> packet option Deferred.t *)
let read_response_packet reader =
  match reader_header reader with
  | Some header -> read_body header reader
  | None -> None
{% endhighlight %}

Like before, there's a slight mismatch between Async and bitstring, and we have to convert between string and bitstring.

Finally, we can now write our small sample program again which connects to a local Memcached server and reads the key "foo":

{% highlight ocaml %}
let main () =
  let host_and_port = Tcp.to_host_and_port "localhost" 12211 in
  Tcp.connect host_and_port >>> fun (_, reader, writer) ->
  let req = make_Request_packet 0 "foo" "" Bigstring.empty_bitstring in
  write_packet writer req >>= fun () ->
  read_packet reader >>= function
    | Some packet when packet.header.status = 0 ->
      Format.printf "Found value: %s\n" packet.value
    | Some packet ->
      Format.printf "Key not found"
    | None ->
      Format.printf "Request failed"
  ;
  Writer.close writer >>= fun () ->
  Reader.close reader
in
main ();
Scheduler.go ()
{% endhighlight %}

### Recap

So what did we gain by rewriting our IO to use Core/Async rather than the built-in standard library? In short, our Memcached client library will play nice with other Async code and will scale much more nicely, e.g. if being used as part of a web server based on Async. As noted a couple of times during the rewrite, Async and bitstring doesn't play very well together. We both need to allocate temporary strings and convert between string and bitstring in multiple places. In a future post I'll discuss other ways of using Reader/Writer which is more efficient.
