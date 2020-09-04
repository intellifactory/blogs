---
title: "News from WebSharper 3.0: NuGet, Visual Studio, Mono, upcoming features and API changes"
categories: "mono,nuget,javascript,f#,websharper"
abstract: "The first batch of WebSharper 3.0 releases has just hit NuGet, together with the Visual Studio extension. The build system has been updated to build correctly on Mono. Work has also started on new features and API enhancements."
identity: "4128,77574"
---
The community response to the WebSharper 3.0 announcement has been overwhelmingly positive. Thanks to everyone who keeps spreading the word! Here are some news from the front:

 * WebSharper 3.0.1.73-alpha just hit [the NuGet repository](http://www.nuget.org/packages/WebSharper/), together with all the public extensions. Don't forget the `-pre` flag when running NuGet update, since they are marked as alpha.

 * We also uploaded the first version 3.0 of the [Visual Studio extension](http://websharper.com/downloads), together with a few updates to the [WebSharper website](http://websharper.com).

 * Thanks to some updates to IntelliFactory.Build and WebSharper itself, WebSharper 3.0 can now be built [from source](https://github.com/intellifactory/websharper) on Mono. Don't hesitate to try it out and report any issues you might encounter!

 * Work has also started on some API enhancements and simplifications and some new features:

    * We are implementing [JavaScript source maps](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/). This feature will allow you to see the original F# code in your browser's debug window, for a much better runtime debugging experience.

    * We are working on enhancing the foreign function interface, in particular regarding higher-order functions. Such functions are currently wrapped at the language interface in a helper that can take arguments either separately (for JavaScript calls) or as a tuple (JavaScript array, for F# calls); but this causes problems eg. with functions taking a single array as argument. We are still exploring solutions.

    * The branch [`js-stdlib-merge`](https://github.com/intellifactory/websharper/tree/js-stdlib-merge) contains a simplification of the bindings for the standard JavaScript library. The gist of it is that everything that is built in the browser is currently spread between no less than five namespaces and modules; this change moves all of it into `IntelliFactory.WebSharper.JavaScript`. More detail in [this commit message](https://github.com/intellifactory/websharper/commit/b269192e2ea11eebc67bac22551f0995319d7c56).

    * We are looking to rename the namespaces `IntelliFactory.Html` and `IntelliFactory.WebSharper.Html` to something more explicit. Many developers have been confused by this naming because the difference is not obvious: the former is for server-side markup, while the latter is for client-side markup. The server-side markup will probably be renamed to `IntelliFactory.WebSharper.Sitelets.Html`; we haven't yet decided a name for the client-side markup.

    * More enhancements will be coming; we hope to have the bulk of the breaking changes in before the end of the year.

Happy coding!
