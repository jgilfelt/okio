Okio
====

A modern I/O API for Java

### java.io is clumsy.

That said, layering streams to transform data is wonderful: decompress, decrypt,
and frame.

But it’s lame that we need additional layers to consume our data: why do we need
`InputStreamReader`, `BufferedReader`, `BufferedInputStream` and
`DataInputStream`? None of these transform the stream; they just give us the
methods we need to consume our data.

Typical implementations of `InputStream` and `OutputStream` hold independent
buffers that need to be sized and managed. Data is copied between buffers at
every step in the chain. For example, a web server serving a JSON HTTP response
makes a long journey from application code to the socket, stopping in several
buffers along the way:

 * Application data
 * JSON encoder
 * Character encoder
 * HTTP encoder
 * Gzip compressor
 * Chunk encoder
 * TLS encryptor
 * Socket

Many of these layers do their buffering with byte arrays and char arrays.
Creating buffers is somewhat expensive in Java because `new byte[8192]` requires
the runtime to zero-fill the memory before it’s readable it to the application.
When we’re only sending 128 bytes of JSON, buffer creation and management can be
a substantial part of the overall workload.

Other `java.io` problems:

 * No timeouts.
 * Many methods are badly-specified: `mark()` / `reset()`, `skip()`, and
   `available()` all have significant flaws.
 * Inefficient defaults. The single-byte `read()` and `write()` methods are
   traps.
 * Type system mismatches. `InputStream.read()` reads a byte (typically
   `-128..127`) but returns an int in `-1..255`.

### java.nio is worse

Programming with NIO is just plain painful. The `Buffer` classes are not safe
for encapsulation: you can’t share your buffer instances with other code without
careful contracts on how the `pos`, `mark`, and `limit` indices will be
manipulated.

NIO buffers are fixed-capacity, so you have to allocate for the worst case or
resize manually.

And then there’s the `Selector` + `Channel` torture that allows you to scale to
very many simultaneous connections.

The most common way to tame NIO is to layer something like [Netty](1) on top.
But Netty adds its own complexity and costs.

### Introducing Okio

We do better by starting with better abstractions. Okio is inspired by
`java.io`, but has some important changes:

 * Buffers that grow (and shrink) on demand. The central datatype `OkBuffer` a
   byte sequence that reads from the front and writes to the back. No `flip()`
   nonsense like NIO!
 * Moving data from one buffer to another is cheap. Instead of copying the bytes
   between byte arrays, `OkBuffer` reassigns internal segments in bulk.
 * No distinction between byte streams and char streams. It’s all data. Read and
   write it as bytes, UTF-8 strings, big-endian 32-bit integers, little-endian
   shorts; whatever you want.  No more `InputStreamReader`.

The package replaces `InputStream` with `Source`, and `OutputStream` with `Sink`.
Every buffer is also a source and a sink, so you don’t need a
`ByteArrayInputStream` and `ByteArrayOutputStream`.

 [1]: http://netty.io/