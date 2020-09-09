---
title: "An F# Web Server From Sockets and Up"
categories: "f#,sockets,performance,webserver"
abstract: "I have implemented a simple web server in F#.  The idea was to try to marry .NET asynchronous socket operations with F# async. **Result**: F# async seems to be the right tool for the job of webserver implementation: it makes asynchronous programming intuitive without adding too much performance overhead."
identity: "2071,74903"
---
I have implemented a simple web server in F#. The idea was to try to marry .NET asynchronous socket operations with F# `async`. **Result**: F# `async` seems to be the right tool for the job of webserver implementation: it makes asynchronous programming intuitive without adding too much performance overhead. The server executes 3500 keep-alive or 1000 normal request per second on my Core i3 machine, compared to 2500/500 requests per second using IIS or `System.Net.HttpListener`.

## Asynchronous Socket Operations

Working with sockets in .NET is done with the `Socket` class. From the MSDN documentation, the recommended approach is to use the asynchronous methods such as `AcceptAsync`, `SendAsync` and `ReceiveAsync`. These methods register callbacks to be executed when data arrives or ships through the socket. As a result of the callback approach, no threads are blocked by slow connections.

## Sockets and F# Async

Unfortunately, the default interface is not very intuitive. The example code is atrocious. Since the operations are callback-based, this seems like a good match for F# `async`. I went for the first mapping that came to mind:

```fsharp
open System.Net.Sockets

/// Thrown when sockets encounter errors.
exception SocketIssue of SocketError

/// Performs AcceptAsync.
val Accept : Socket -> Async<Socket>

/// Performs ReceiveAsync.
val Receive : Socket -> System.ArraySegment<byte> -> Async<int>

/// Performs SendAsync.
val Send : Socket -> System.ArraySegment<byte> -> Async<unit>
```

Implementing this interface is easy - it is just working around the boilerplate: creating a `SocketAsyncEventArgs` object, registering a callback, calling the method, checking the result for errors. I was able to express all of it in a single helper method:

```fsharp
open System.Net.Sockets

type A = System.Net.Sockets.SocketAsyncEventArgs
type B = System.ArraySegment<byte>

exception SocketIssue of SocketError with
    override this.ToString() =
        string this.Data0

/// Wraps the Socket.xxxAsync logic into F# async logic.
let inline asyncDo (op: A -> bool) (prepare: A -> unit)
    (select: A -> 'T) =
    Async.FromContinuations <| fun (ok, error, _) ->
        let args = new A()
        prepare args
        let k (args: A) =
            match args.SocketError with
            | System.Net.Sockets.SocketError.Success ->
                let result = select args
                args.Dispose()
                ok result
            | e ->
                args.Dispose()
                error (SocketIssue e)
        args.add_Completed(System.EventHandler<_>(fun _ -> k))
        if not (op args) then
            k args

/// Prepares the arguments by setting the buffer.
let inline setBuffer (buf: B) (args: A) =
    args.SetBuffer(buf.Array, buf.Offset, buf.Count)

let Accept (socket: Socket) =
    asyncDo socket.AcceptAsync ignore (fun a -> a.AcceptSocket)

let Receive (socket: Socket) (buf: B) =
    asyncDo socket.ReceiveAsync (setBuffer buf)
        (fun a -> a.BytesTransferred)

let Send (socket: Socket) (buf: B) =
    asyncDo socket.SendAsync (setBuffer buf) ignore
```

## Optimizations

It seems that the common optimization paths include pooling sockets, pooling `SocketAsyncEventArgs`, and pooling buffers to prevent memory fragmentation. The latest point is the most interesting. Socket code itself is written in unmanaged C and passing data between garbage-collected .NET code and C code is done by *pinning* the .NET array used as a buffer. A pinned array is never relocated by the garbage collector, so the C code has no trouble finding it. A lot of pinned arrays to work around make garbage collector's job harder - memory gets fragmented.

To avoid fragmentation issues, instead of allocating a lot of small arrays I allocate one huge array and then lease sections of it to be used as buffers by the socket operations.

I have not yet tried pooling `Socket` or `SocketAsyncEventArgs` objects in a similar manner.

## Benchmarks

For benchmarking I have used Apache Bench (ab) tool running on Arch Linux inside a VirtualBox VM. All benchmarks involved dynamically generating and serving a "HELLO, WORLD" document on my Core i3 laptop, with `ab -k -c 1000 -n 10000`:


Server| Keep-alive r/s | Regular r/s
:-----|:-----|:---------|
F# WebServer | 3500 | 1000 |
Haskell warp/wai GHC 7 | 3500 | 3500 |
IIS | 2500 | 500 |
System.Net.HttpListener | ? | 500 |
node.js (Windows) | 800 | 400 |
node.js (Linux) | ? | 3000 |


I do not feel very good about these numbers, in particular because I have seen claims of Haskell WARP doing 90000 r/s on only slightly faster hardware (8-core Core i5). It may be that I am hitting VirtualBox networking overhead or I have not built the Haskell code with proper flags.

But for what they are worth, the numbers seem to indicate that F# async is a good enough foundation for web servers with performance in the IIS league. It does not need to be faster, it just needs to be good enough. The real advantage is that F# async code is tremendously easier to read and write than explicit callback code.

**EDIT**: Please do take the benchmarks with a grain of salt. They are far from comprehensive or correctly done.
