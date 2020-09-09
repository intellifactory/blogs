---
title: "Upcoming WebSharper Changes"
categories: "websharper,f#"
abstract: "Soon-to-be released WebSharper includes Direct API that will greatly simplifies building tools that compile F# code to JavaScript via WebSharper. It has can work in FSI. We are also finishing some testing on a TypeScript definition file cross-compiler that emits WebSharper FFI definitions based on TypeScript \".d.ts\" files, allowing to reuse some work the TypeScript community has done in defining types for JavaScript libraries."
identity: "3053,76214"
---
It is time for some long-awaited improvements in WebSharper: Direct API and TypeScript cross-compiler. 

## Direct API

While working on the [CloudSharper (WebSharper IDE)](https://www.facebook.com/media/set/?set=vb.189105887856115&type=2) project, we strongly felt the need to have a more flexible way to call into WebSharper, to have the compiler as a service. In particular, we wanted it to be able to compile code on the fly inside an FSI session, including parsing FSI-defined dynamic assemblies.  While very reasonable in retrospect, this requirement was not on the table in the original design, and we had to put in quite  a few changes to make it work.  The good news is that it is working, and you can even compile F# to JavaScript inside an FSI session.  The changes are already published in our [official WebSharper repository](https://github.com/dotnet-websharper/core)[^1], in particular see [FrontEnd.fsi](https://github.com/dotnet-websharper/core/blob/master/src/compiler/WebSharper.Compiler/FrontEnd.fs)[^2].  I am now finalizing some cleanup tasks and will soon release a NuGet package update incorporating the changes.

Direct API makes a few frequently requested scenarios much easier. For example, it is now a lot more straightforward to avoid reliance on ASP.NET, to build more flexible tools, or to do things like targeting Node and a pure-JavaScript runtime.

## TypeScript

In case you have not noticed, there is a new language from Microsoft called [TypeScript](https://www.typescriptlang.org).  After working with deciphering its semantics for a bit, I am not a big fan of TypeScript design (this deserves another blog); but I will grant that it is a vast improvement on JavaScript.  Moreover, TypeScript defines a standard way to describe JavaScript library interfaces, and people have started doing so en masse, see for example [DefinitelyTyped](https://github.com/borisyankov/DefinitelyTyped).  With the TypeScript cross-compiler tool we are trying to reuse all this work - and generate WebSharper FFI bindings based on TypeScript ".d.ts" files, using either a F# 3.0 TypeProvider or bulid-time API.  A competing project, [FunScript](https://github.com/ZachBray/FunScript), has pioneered the approach.

TypeScript bindings are likely to be the future of WebSharper extensions story, even though the quality of the present TypeScript bindings strikes me as very poor. Well, so does the quality of TypeScript definition language. TypeScript does not, for instance, allow generics in the specifications, though they are considering to add them in the next release.  Regardless, the reason that TypeScript bindings are exciting is social - due to excellent tooling and Microsoft backing TypeScript is picking up very quickly. There are already many more TS bindings than WebSharper bindings, and given the dynamic of its adoption, their quality is likely to improve quickly. We would like to ride the wave.


[^1]: This link has been updated to point to new location.

[^2]: This link has been updated to point to new location.
