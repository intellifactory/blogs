---
title: "Compiling to JavaScript dynamically with WebSharper"
categories: "f#,c#,websharper,compiler,fsadvent"
abstract: ""
identity: "-1,-1"
---

There are two big community projects existing for transpiling F# code to JavaScript: WebSharper and Fable. Hovewer, the focus of the two project is somewhat different, here is how I see it:

* WebSharper has bigger focus on living inside .NET ecosystem, while Fable has bigger focus of integrating output into JavaScript ecosystem as well as other target languages.
  * WebSharper supports C#, and with it correct semantics for all object-oriented .NET features of F# like inheriting from multiple base constructors, lazily executing static constructors. Code interop between C# and F# works as in .NET. Array operations do bound checks, F# option values are not erased, so you have maximum semantic compatibility with .NET and it's your choice to optimize with `JavaScript.Array` or `Optional` (erased option) types when needed for interop. Also, you can have server and client code living inside same project, encapsulating whole server-client functionality and JavaScript dependencies that can be distributed via NuGet. 
  * Fable has bigger focus on code output interoperability, while sacrificing some .NET and F# object-oriented features and semantics. It uses F#'s parser and type system to analyze F# source code that are intended for Fable translation only, or using compiler directives to diverge between Fable and .NET code. It distributes source code files via NuGet, not compiled dlls, and can depend on npm packages.
* WebSharper has tightly integrated metaprogramming features.
  * You can write macros and code generators within your libraries by inheriting from `WebSharper.Core.Macro` and `Generator` types. Macros allow custom translation logic for annotated methods/types, while generators can create JS code output from some logic executing in compile-time that returns a JS string, F# quotation or WebSharper AST. For example macros allow for auto-implemented [two-way lensing](https://intellifactory.com/user/denuziere/20180228-clear-and-simple-reactive-code-with-websharper-ui-s-v) in `WebSharper.UI`.

WebSharper's main shortfall is that its output can be pretty crude and does not integrate with module systems and bundlers well enough. The original intention was that your main web project is a .NET project that can handle everything for you, this legacy got carried along and there was always more focus on improvements on the input side (C# support, more language and framework features, etc.) than on the output side. [WebSharper 7](https://github.com/dotnet-websharper/core/pull/1302) is under development now that will bring output up to modern JavaScript standards with module output, JS classes, get/set properties, arrow functions and more.

## How to use compiler dynamically

While this update is under way, let us look at a minimal version of [Try WebSharper](https://try.websharper.com/), a simple web project that exposes JS translations of either F# or C# source code input on a minimal UI. This will eventually be a good tool to demo WebSharper 7's improved output.

The project is available on [Github](https://github.com/Jand42/CompileJSLive).

Clone and run it with:

```
git clone https://github.com/Jand42/CompileJSLive
cd CompileJSLive
dotnet run
```

The project uses both the `WebSharper.Compiler.FSharp` and `WebSharper.Compiler.CSharp` packages, which are containing the WebSharper compiler libraries as references to be used by our code, not as tools (which are used for translating WebSharper-enabled projects but not invoking the compiler).
These depend on `FSharp.Compiler.Services` and `Microsoft.CodeAnalysis.CSharp` (Roslyn) respectively, so we don't need to directly reference these.

WebSharper uses metadata embedded in dlls to guide its translation to conform to already translated projects with embedded JS files in them. Some of this metadata is used for the server-side runtime of web projects to enable client-server features of websharper for example creating controls on the server which will be filled in on the client with content (the `client` helper functions in `Site.fs`). Usually this runtime metadata does not need to contain any code expressions, just the overall shape of the generated JS so that event handlers and client-side placeholders can refer to it. But for on-the-fly translation, we also need the inline expressions from metadata, which are expanded upon use. This can be made accessible with a single new option in `wsconfig.json`: `"runtimeMetadata": "inlines"`.

Now the server-side translation of both F# and C# are pretty similar in this sample:

* Use FCS/Roslyn to create a Compilation object from the given source code, using a list of hardcoded references for simplicity
* Initialize WebSharper's Compilation object from the F#/C# compilation results and give it the metadata extracted from current remoting context
* Compile it, package it to a single expression, write to JS and return, or optionally return list of errors if anything went wrong

The project showcases how to do this simply. Note however that the Compilation has a `UseLocalMacros = false` constructor argument, this is guarding against executing user-submitted code via macro functionality. Without this, macros would still not run, as we would need to load a dynamically compiled assembly created from the source into an assembly load context, this argument is just making sure WebSharper does not even attempts to look up macros from current project and gives a specific error about it.

This is a small project for playing around, there are many complexities, but I hope I could give an understandable but very concise summary.

Contributors are welcome on WebSharper, feel free to send me a message on [F# Software Foundation Slack](https://fsharp.slack.com/). 

### Happy Holidays! Merry Christmas!
