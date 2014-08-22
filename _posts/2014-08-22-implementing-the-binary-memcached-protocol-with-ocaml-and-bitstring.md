---
layout: post
title: Implementing the Binary Memcached Protocol with Ocaml and Bitstring
---

I've known Ocaml for a while, but I never really put it to use outside of academic work back in university. After reading [Real World Ocaml](https://realworldocaml.org/) my excitement for Ocaml reappeared and I wanted to try out [Core](https://ocaml.janestreet.com/ocaml-core/latest/doc/) and [Async](https://realworldocaml.org/v1/en/html/concurrent-programming-with-async.html). At the back of my mind, I also remembered [bitstring](https://code.google.com/p/bitstring/), an awesome Ocaml library for creating and pattern matching on bitstrings. To cover all three, I decided to write a Memcached client using the [binary protocol](https://code.google.com/p/memcached/wiki/MemcacheBinaryProtocol).

I'll try to cover my experience in a number of blog posts. In this first post, I'll be talking about using bitstring for parsing and constructing binary data for the Memcached protocol.

### Constructing Requests and Parsing Responses

I started out by studying the [Memcached binary protocol](https://code.google.com/p/memcached/wiki/MemcacheBinaryProtocol). The general format of a packet is a fixed size header followed by three optional variable size components (command-specific extras, the key and the value):

```
Byte/     0       |       1       |       2       |       3       |
   /              |               |               |               |
  |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
  +---------------+---------------+---------------+---------------+
 0/ HEADER                                                        /
  /                                                               /
  /                                                               /
  /                                                               /
  +---------------+---------------+---------------+---------------+
24/ Command-specific extras (as needed)                           /
 +/  (note length in the extras length header field)              /
  +---------------+---------------+---------------+---------------+
 m/ Key (as needed)                                               /
 +/  (note length in key length header field)                     /
  +---------------+---------------+---------------+---------------+
 n/ Value (as needed)                                             /
 +/  (note length is total body length header field, minus        /
 +/   sum of the extras and key length body fields)               /
  +---------------+---------------+---------------+---------------+
```

The header is almost similar for requests and responses and can be described in the following way:

```
Request header:

Byte/     0       |       1       |       2       |       3       |
   /              |               |               |               |
  |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
  +---------------+---------------+---------------+---------------+
 0| Magic         | Opcode        | Key length                    |
  +---------------+---------------+---------------+---------------+
 4| Extras length | Data type     | Reserved                      |
  +---------------+---------------+---------------+---------------+
 8| Total body length                                             |
  +---------------+---------------+---------------+---------------+
12| Opaque                                                        |
  +---------------+---------------+---------------+---------------+
16| CAS                                                           |
  |                                                               |
  +---------------+---------------+---------------+---------------+
  Total 24 bytes

Response header:

Byte/     0       |       1       |       2       |       3       |
   /              |               |               |               |
  |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
  +---------------+---------------+---------------+---------------+
 0| Magic         | Opcode        | Key Length                    |
  +---------------+---------------+---------------+---------------+
 4| Extras length | Data type     | Status                        |
  +---------------+---------------+---------------+---------------+
 8| Total body length                                             |
  +---------------+---------------+---------------+---------------+
12| Opaque                                                        |
  +---------------+---------------+---------------+---------------+
16| CAS                                                           |
  |                                                               |
  +---------------+---------------+---------------+---------------+
  Total 24 bytes
```

Let's go through the header fields one by one:

- `Magic` is a magic byte which distinguishes requests from responses. It must be `0x80` for requests and `0x81` for responses.
- `Opcode` specifies the type of command, e.g. `GET` is `0` and `SET` is `1`.
- `Key Length` specifies the size in bytes of the `Key`-part of the packet.
- `Extras length` specifies the size in bytes of the `Command-specific extras`-part of the packet.
- `Data type` is currenly not used and should always be `0`.
- Bytes 7-8 is either `Reserved` in the request or `Status` in the response. The status corresponds to an error code or `0` if successful.
- `Total body length` is the total size of the body in bytes, i.e. `Total body length = Key length + Key length + Value length`. The value length is not given explicitly in the header, but can be calculated as `Total body length - Key length - Extras length`.
- `Opaque` is 4 bytes which are passed along in a request and passed back unmodified in the response.
- `CAS` is used for data version checking. When storing a value you can optionally set its CAS value. Future modifications will be rejected if a matching CAS value is not provided.

As you can see, the only real difference between a request and a response header is that bytes 7-8 are reserved in the request header and used for status in the response header. Ignoring this difference, we can define a unified type for Memcached headers:

{% highlight ocaml %}
type header = {
  opcode:        int;   (* maps to Memcached commands, e.g. GET is 0 *)
  key_length:    int;   (* length of the key component *)
  extras_length: int;   (* length of the extras component *)
  status:        int;   (* should not be used in requests *)
  body_length:   int32; (* total length of the body *)
  opaque:        int32; (* is returned unmodified in the response *)
  cas:           int64; (* used for data version checking *)
}
{% endhighlight %}

I've omitted data type, since it's currently not used as noted above.

Now that we have a header type, we just need to figure out how to marshal and unmarshal it from binary data. We'll use the [bitstring](https://code.google.com/p/bitstring/) library for this purpose. It provides a very convenient way to construct binary data and pattern match on binary data, reminiscent of [bit syntax in Erlang](http://www.erlang.org/documentation/doc-5.6/doc/programming_examples/bit_syntax.html). Given a header, we can construct a bitstring like so:

{% highlight ocaml %}
(*  header_to_bitstring : header -> bitstring *)
let header_to_bitstring h =
    BITSTRING {
      0x80            :  8;
      h.opcode        :  8;
      h.key_length    : 16;
      h.extras_length :  8;
      0               :  8;
      h.status        : 16;
      h.body_length   : 32;
      h.opaque        : 32;
      h.cas           : 64
    }
{% endhighlight %}

`BITSTRING { ... }` will return a value of type `bitstring`. Each line inside the braces has the form `value : size`. The first line thus says *the first 8 bits should hold the value 0x80* (the magic byte for requests).

Given a bitstring we can similarly parse it to a header in the following manner using the `bitmatch` operator:

{% highlight ocaml %}
(*  header_of_bistring : bitstring -> header option *)
let header_of_bitstring bits = bitmatch bits with
  | { 0x81          :  8 ;
      opcode        :  8 ;
      key_length    : 16 : unsigned;
      extras_length :  8 : unsigned;
      0             :  8 ;
      status        : 16 ;
      body_length   : 32 : unsigned;
      opaque        : 32 ;
      cas           : 64
    } -> Some {
           opcode;
           key_length;
           extras_length;
           status;
           body_length;
           opaque;
           cas;
         }
  | { _ } -> None
{% endhighlight %}

`bitmatch bits with [patterns]` will match `bits` of type `bitstring` against each pattern, which follows the same syntax as for `BITSTRING`. As you can tell, it's much simpler to build and parse binary strings with bitstring compared to manual bit fiddling!

Both of these functions work with the type `bitstring` defined in the bitstring library.  So how do we read a bitstring from IO or write a bitstring to IO? The bitstring library defines a number of functions which interact with channels (channels are the IO abstraction in the standard library):

{% highlight ocaml %}
(* read all of a chan to a bitstring *)
bitstring_of_chan     : in_channel -> bitstring

(* read at most n bytes of a chan to a bitstring *)
bitstring_of_chan_max : in_channel -> int -> bitstring

(* write a bitstring to a chan *)
bitstring_to_chan     : bitstring -> out_channel -> unit
{% endhighlight %}

Given those, it should be simple to parse a header from a channel:

{% highlight ocaml %}
(*  read_header : in_channel -> header *)
let read_header channel =
  let bits = Bitstring.bitstring_of_chan_max channel 24 in
  header_of_bitstring bits

(*  write_header : out_channel -> header -> unit *)
let write_header channel header =
  let bits = header_to_bitstring header in
  Bitstring.bitstring_to_chan bits channel
{% endhighlight %}

At this point we can read and write Memcached headers to and from channels. To handle complete Memcached packets, we need to consider the variable sized body too. Let's define a packet type:

{% highlight ocaml %}
type packet = {
  header : header;
  extras : bitstring;
  key    : string;
  value  : string;
}
{% endhighlight %}

The key and value are strings, while the extras contain binary data (a bitstring). We're now ready to read and write entire packets from channels. Here's the code:

{% highlight ocaml %}
(*  make_request_packet : int -> string -> string -> bitstring -> packet *)
let make_request_packet opcode key value extras =
  let extras_length = (Bitstring.bitstring_length extras)/8                    in
  let key_length    = String.length key                                        in
  let value_length  = String.length value                                      in
  let body_length   = Int32.of_int (extras_length + key_length + value_length) in
  let header = {
    opcode;
    key_length;
    extras_length = extras_length;
    status = 0;
    body_length = body_length;
    opaque = 0l;
    cas = 0L;
  } in
  { header; extras; key; value }

(* write_packet : out_channel -> packet -> unit *)
let write_packet channel packet =
  let header_bits = header_to_bitstring packet.header in
  Bitstring.bitstring_to_chan header_bits channel;
  Bitstring.bitstring_to_chan packet.extras channel;
  output_string channel packet.key;
  output_string channel packet.value

(*  read_body : header -> in_channel -> packet option *)
let read_body header channel =
  let body_length  = Int32.to_int header.body_length                        in
  let bits         = Bitstring.bitstring_of_chan_max channel body_length    in
  let value_length = body_length - header.extras_length - header.key_length in
  bitmatch bits with
    | { extras : 8*header.extras_length : bitstring;
        key    : 8*header.key_length    : string;
        value  : 8*value_length         : string
      } -> Some { header; extras; key; value; }
    | { _ } -> None

(* read_response_packet : in_channel -> packet option *)
let read_response_packet channel =
  match read_header channel with
  | Some header -> read_body header channel
  | None -> None
{% endhighlight %}

As a trivial example, we can now send a `GET` request for the key `foo` to a Memcached server running on localhost port 11211:

{% highlight ocaml %}
let localhost         = Unix.inet_addr_of_string "127.0.0.1"                     in
let addr              = Unix.ADDR_INET (localhost, 11211)                        in
let in_chan, out_chan = Unix.open_connection addr                                in
let req               = make_request_packet 0 "foo" "" Bitstring.empty_bitstring in
write_packet out_chan req;
flush out_chan;
match read_response_packet in_chan with
  | Some packet when packet.header.status = 0 ->
      Format.printf "Found value: %s\n" packet.value
  | Some packet ->
      Format.printf "Key not found"
  | None ->
      print_endline "Request failed";
Unix.shutdown_connection in_chan
{% endhighlight %}

Obvious this is still a very low level interface, but it's easy to build on top of the building blocks we've implements. Defining functions with a simpler signature to get and set keys should be easy, but I'll leave that as an exercise for the reader.

### Next time...

In this blog post we've developed a primitive Ocaml library for talking to Memcached with the binary protocol using [bitstring](https://code.google.com/p/bitstring/). The code is based on the standard library, which is honest not that great, and all IO is synchronous. In the next installment we'll try using [Core](https://ocaml.janestreet.com/ocaml-core/latest/doc/) and [Async](https://realworldocaml.org/v1/en/html/concurrent-programming-with-async.html) for asynchronous IO. Stay tuned!
