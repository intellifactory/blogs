---
title: "WebSharper available for testing on Xamarin & MonoDevelop"
categories: "f#,websharper,monodevelop,mono,xamarin"
abstract: "WebSharper development gains support for Mono framework on Linux and Mac OS X, and integration with Xamarin Studio and MonoDevelop."
identity: "3803,77123"
---
WebSharper development gains support for Mono framework on Linux and Mac OS X, and integration with Xamarin Studio and MonoDevelop. The preview add-in is available for testing at [dotnet-websharper/monodevelop-websharper](https://github.com/dotnet-websharper/monodevelop-websharper)[^1].

What you get:

 * Ability to develop WebSharper projects on Linux and Mac OS X
 * A viable IDE alternative to Visual Studio for WebSharper projects

What we have done:

 * Worked around a bunch of xbuild bugs by rewriting WebSharper build logic from MSBuild XML to MSBuild tasks, this seems to work reliably on both Mono/xbuild and .NET/MSBuild
 * Created an add-in for MonoDevelop and Xamarin Studio that is able to create instances of WebSharper templates

There are some remaining issues with Web project support, especially F#/Web projects integration into the IDE, but the projects are working with **xsp4**.

In the near future, we plan to push this further and hope to fully support cross-platform development of OWIN-based Web projects in F#/WebSharper that would allow hosting performant websites directly in-process, rather than relying on ASP.NET xsp4 emulation.


[^1]: This link has been updated to point to new location.
