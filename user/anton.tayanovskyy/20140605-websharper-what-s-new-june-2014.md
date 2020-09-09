---
title: "WebSharper: What's New (June 2014)"
categories: "f#,websharper,spa,frontend"
abstract: "WebSharper is getting support for Mono, MonoDevelop, Xamarin Studio, CloudSharper on Linux and Mac OS X, improved bindings to D3.js and Leaflet.js. On performance side, we are experimenting with ASM.js for numeric code. For this summer, there are plans for better single-page application support and a functional version of data binding, to incorporate and improve on ideas in Angular JS and Facebook React."
identity: "3903,77242"
---
## Mono, Mac and Linux

Since February, we worked on supporting WebSharper development, and not just deployment, on Mono, Mac and Linux. Most of the code works fine, but unfortunately Mono support for MSBuild with XBuild leaves much to be desired, and Xamarin team is in no hurry to fix or respond to this - see https://bugzilla.xamarin.com/show_bug.cgi?id=18761 and https://bugzilla.xamarin.com/show_bug.cgi?id=15325 - as they do not see XBuild/MSBuild compatibility as a priority.

Fortunately, we were able to find workarounds to avoid buggy XBuild features, by moving most build logic to F# code invoked via XBuild/MSBuild plugins. There is now some preliminary support for building on Mono and using MonoDevelop or Xamarin Studio 4.2, with 5.0 support and project turnaround between all IDEs (Visual Studio, XS/MD, CloudSharper) coming soon.

## Numerics

We are working with Microsoft Research to improve WebSharper performance on numeric code for applications such as numeric biology. As part of this work, we created an experimental compiler from an F# subset to ASM.js. So far, the prototype failed to provide speedups on current benchmarks, but came quite close. As the experiments continue, we are considering more approaches, such as integrating loop-invariant code motion into the optimizer, or perhaps even taking advantage of LLVM IR and existing passes written for it. 

## Visualization and Extensions

 * Andras Janko has done [interesting work](http://fpish.net/blog/JankoA/id/3866/2014520-sencha-architect-type-provider-for-websharper-update) lowering the barrier to using Sencha Architect UI design tool from WebSharper with Type Providers.

 * [D3.js](https://github.com/dotnet-websharper/d3)[^1] binding received an update and multiple improvements. There are also now a few more [samples](https://websharper-samples.github.io/D3/)[^2] of using this wonderful library with WebSharper.

 * [Leaflet.js](https://github.com/dotnet-websharper/leaflet)[^3] now also has a WebSharper binding.


## Single-Page Applications and Reactive DOM

There are some lively discussions on better SPA support with the IntelliFactory team - Andras Janko, Loic Denuziere and Simon Fowler who is joining us for a summer internship. We are evaluating Facebook React, Angular JS, and D3.js and looking how we can do better in a typed language, in particular how to better integrate dataflow and data binding. If the prototypes work out, WebSharper is likely to obtain a new powerful front-end combinator library by the end of this summer.


[^1]: This link has been updated to point to new location.

[^2]: This link has been updated to point to new location.

[^3]: This link has been updated to point to new location.
