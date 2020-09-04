---
title: "WebSharper 2.5.134 released"
categories: "f#,websharper"
abstract: "Introducing proxies for Async cancellation, MailboxProcessor, Printf."
identity: "4123,77566"
---
A lot of useful functions from `FSharp.Core` are now translated by WebSharper.

## Async cancellation

 * Proxies for `CancellationTokenSource`, `CancellationToken`, `CancellationTokenRegistration`.
 * Proxies for `Async` class members: `DefaultCancellationToken`, `CancelDefaultToken`, `CancellationToken`, `OnCancel`, `TryCancelled`
 * Async RPC calls are cancellable, the cancel continuation runs ASAP and the response are discarded.


## MailboxProcessor

 * Proxy handles timeout arguments and cancellation
 * `PostAndReply`/`TryPostAndReply` methods are not supported as JS is single-threaded.


## Printf

 * Proxies for `sprintf, printfn, failwithf, Printf.kbprintf`
 * `%O` uses JavaScript `String(x)` function
 * `%A` generates a recursive pretty-printer for F# types if type information is available. Currently supports lists, unions, tuples, records, 1 and 2 dimensional arrays. Other types are sent to a dynamic pretty-printer.
 * `printfn` prints to JavaScript console
 * `Printf.sprintf` is not supported, use `ExtraTopLevelOperators.sprintf` instead.
 * Printing as unsigned is not supported.


## Fixes

 * Fix [#300](https://bitbucket.org/IntelliFactory/websharper/issue/300): Server-side runtime error in some regional settings (for example Turkish)
 * Fix [#304](https://bitbucket.org/IntelliFactory/websharper/issue/304): Converting string to char checks string length now
 * `Async.FromContinuations` ensures that continuation is called only once, throws an error otherwise.


## Breaking changes

 * `System.Web.HttpPostedFile` type in Sitelets API [have been changed](https://bitbucket.org/IntelliFactory/websharper/issue/298/) to `System.Web.HttpPostedFileBase` to enable non-ASP.NET hosting (OWIN). 
 * Indexed properties in WIG: [Previously](http://fpish.net/blog/JankoA/id/4015/201494-websharper-2-5-124-61-released) if a property was defined with name `"item"`, it translated to a an indexer on the object. This is changed so that the empty string is usable for this (still will get .NET property name Item, the default indexer for the class).


    ```fsharp
    // generates default .NET indexer
    "" =@ T<obj> |> Indexed T<int> // has JavaScript inline "$this[$index]"
    ```
