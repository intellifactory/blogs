---
title: "WebSharper - a year in review"
categories: "livedata,jointjs,rappid,forms,review,react,bootstrap,c#,f#,websharper"
abstract: "Just over a year ago, last year in December we released WebSharper 3 on Apache, putting it into the hands of every F# web developer. One thing we learned from WebSharper 2 is that more frequent releases are better and this year kept the whole team busy with constant innovation and new releases. Below is a list I cherry-picked from the WebSharper blog.. [more]"
identity: "4665,81029"
---
## WebSharper – a year in review

Just over a year ago, last year in December [we released WebSharper 3 on Apache](/user/granicz/20141203-websharper-3-alpha-now-under-apache-2.md), putting it into the hands of every F# web developer. One thing we learned from WebSharper 2 is that more frequent releases are better and this year kept the whole team busy with constant innovation and new releases. Below is a list I cherry-picked from [the WebSharper blog](http://websharper.com/blog):
 
 * WebSharper 3.0 with support for [source maps](/user/jankoa/20141216-websharper-3-0-3-alpha-released.md), [embedded sitelets](https://github.com/intellifactory/websharper/issues/307), a [more standardized API](/user/denuziere/20150108-websharper-3-0-8-alpha-published.md) and [namespace changes](/user/jankoa/20150225-websharper-3-0-36-alpha-released.md), better JavaScript interoperability [#1](/user/jankoa/20150210-websharper-3-0-26-alpha-released.md), [#2](/user/jankoa/20150225-websharper-3-0-36-alpha-released.md), [#3](/user/denuziere/20150318-websharper-3-0-rc-released.md), and [#4](/user/jankoa/20150416-websharper-3-0-released.md), [streamlined routing syntax](/user/denuziere/20150213-upcoming-in-websharper-3-0-serving-rest-apis-easy-as-pie.md), [MonoDevelop and Xamarin Studio templates](/user/denuziere/20150225-websharper-3-0-alpha-for-xamarin-studio-monodevelop-is-now-available.md), [self-hosted applications](/user/denuziere/20150506-websharper-3-0-59-released.md)

 * [WebSharper 3.1](/user/denuziere/20150523-websharper-3-1-published.md) with support for ASP.NET MVC, lightweight syntax to embed client-side code into sitelets, more routing enhancements (wildcard paths, multiple endpoints with a shared URL prefix, posting form data)

 * [WebSharper 3.2](/user/granicz/20150609-websharper-3-2-with-support-for-scriptable-applications-better-resource-management-and-additional-streamlined-syntax.md) with streamlined syntax for sitelets, dot syntax for event handlers, server-side templating enhancements, more [router syntax simplifications](/user/jankoa/20150625-websharper-3-2-10-released.md), and [WebSharper.Warp-related enhancements](/user/denuziere/20150714-websharper-3-2-22-released.md)

 * [WebSharper 3.3](/user/denuziere/20150722-websharper-3-3-released-with-client-side-json-serialization.md) with client-side JSON serialization, F# 4.0 and its new collection functions, and cross-tier server-client-reusable HTML.

 * [WebSharper 3.4](/user/denuziere/20150803-websharper-3-4-released.md) with a revamped Sitelets API, [streamlined HMTL syntax](/user/denuziere/20150803-websharper-ui-next-3-4-the-new-html-syntax.md) (lowercase HTML combinators, more natural attribute syntax, cross-tier includes, client-side extensions, basic reactive templating), cross-site RPC for mobile web applications, [new templates](/user/granicz/20150806-new-websharper-templates.md). You also get [ASP.NET-hosted OWIN applications, data binding] (/user/denuziere/20150902-websharper-3-4-14-released.md) and a ton of [other enhancements](/user/denuziere/20150924-websharper-3-4-19-released.md) in UI.Next applications, as well as [support for `OnAfterRender`](/user/denuziere/20150908-websharper-ui-next-3-4-19-with-onafterrender.md).

 * WebSharper 3.5 with [pluggable HTML support](/user/granicz/20151007-announcing-websharper-3-5-with-pluggable-html-support.md), and various fixes [#1](/user/denuziere/20151021-websharper-3-5-9-released.md), [#2](/user/denuziere/20151028-websharper-3-5-13-released.md), [#3](/user/jankoa/20151030-websharper-3-5-14-released.md), and splitting off the internal [testing framework](/user/denuziere/20151112-websharper-3-5-16-released.md) into a separate library

 * WebSharper 3.6 with [CDN support](/user/denuziere/20151123-websharper-3-6-released-with-cdn-support.md), making WebSharper 3.6+ applications blazing fast, and a built-in [cookies library and server-side resources](/user/denuziere/20151215-websharper-3-6-6-released.md).

We have also:

 * [Beefed up the WebSharper website](/user/granicz/20150428-websharper-site-enhancements.md)

 * Showed how to do [auto-refreshing Azure-deployments on GitHub commits](/user/denuziere/20150512-websharper-from-zero-to-an-azure-deployed-web-application.md) and provided a new [template](/user/granicz/20150515-deploying-websharper-apps-to-azure-via-github.md) for these projects

 * [Introduced WebSharper.Warp](/user/granicz/20150615-introducing-websharper-warp.md): a frictionless WebSharper library for building scripted and standalone OWIN-based, full-stack F# web applications.

 * [Introduced Try WebSharper](/user/granicz/20150804-introducing-try-websharper.md), an online mini-IDE for developing and [sharing](/user/granicz/20150809-share-and-embed-try-websharper-snippets.md) WebSharper snippets with on-the-fly type checking, code completion and compiling to JavaScript. You can also [version these snippets, import them from gists](/user/granicz/20150819-try-websharper-snippet-versioning-gist-import-and-other-enhancements-now-available.md) and add [update notes](/user/gansperger/20150826-try-websharper-update-notes-for-snippets.md) to each version. And best of all, you can use almost [all WebSharper extensions](/user/granicz/20150901-live-f-coding-and-snippets-with-dependencies-in-try-websharper.md) and you get [on-hover type info](/user/gansperger/20150918-try-websharper-on-hover-type-info.md) and [extension info](/user/gansperger/20151006-try-websharper-version-info-about-extensions-and-some-embedding-improvements.md) as well.

 * [Introduced WebSharper.Suave](/user/denuziere/20151001-announcing-websharper-suave.md), a middleware for running WebSharper sitelets on Suave, and a new [client-server template](/user/granicz/20151007-announcing-websharper-3-5-with-pluggable-html-support.md) to help you get started easily.

 * [Rolled out WebSharper.Data and WebSharper.Charting](/user/granicz/20151104-data-aware-workbooks-and-client-side-data-access-with-websharper-data.md), making it easy to work with heterogeous data sources from client-side code and adding advanced charts and visualizations to your projects.
 
 * Exceeded 1000 topics on the [WebSharper Forums](http://websharper.com/questions).
 
### Conferences and academia

This year we concentrated on getting work out and attended fewer conferences than in previous years.  In 2014, the WebSharper team gave 16 talks on WebSharper in six countries, this year we gave nine talks in six countries.

The team submitted three research papers to academic conferences in 2015, continuing our tradition to publish research results in peer-reviewed conferences.

Next to conferences and research papers, our team has been active in teaching and trainings.  Among others, we run functional reactive web development courses with F# and WebSharper at the Eotvos Lorand University (ELTE) and the University of Dunaujvaros, and have signed a similar agreement with the University of Szeged.  We are also involved in creating an online semester long course for international students.

### Coming up

Our WebSharper team has been cooking up some awesome things that are not yet available or not yet documented. In particular, with more blog entries coming up on each:

 * **WebSharper 4** - we are finalizing the last bits of moving WebSharper to the F# Compiler Services (FCS) and merging the F#+WebSharper compilation steps into a single sweep, giving a significant boost to compilation speed, better F# language coverage, and fixing a number of corner cases in earlier releases. The first alphas are out in early January under a new WebSharper codename “Zafir”.
 
 * **WebSharper 4 for C#** - we are finally able to bring WebSharper to C# developers, covering the most common C# scenarios (asyncs, LINQ, etc.), most of the language, and calls to any one of the existing extensions in the WebSharper ecosystem. A lot more on this in upcoming blogs.
 
 * **WebSharper.React** - bringing [React](https://facebook.github.io/react) to WebSharper applications. Here is a live example:
 
   <div style="width:100%;min-height:300px;position:relative"><iframe style="position:absolute;border:none;width:100%;height:100%" src="http://try.websharper.com/embed/sandorr/00005G"></iframe></div>

 * **WebSharper.LiveData** - automatic syncronization of reactive data models between a server and its participating clients.  You can read a draft of [our first, upcoming tutorial](https://github.com/Tarmil/websharper.docs/blob/master/tutorials/LiveData.md).

 * **WebSharper.Forms** and **WebSharper.Forms.Bootstrap** - reactive web forms (aka. reactive Piglets) with custom rendering. Here is a live example that uses Bootstrap-based rendering for a login form:
 
   <div style="width:100%;min-height:400px;position:relative"><iframe style="position:absolute;border:none;width:100%;height:100%" src="http://try.websharper.com/embed/adam.granicz/00004x"></iframe></div>
 
 * New extensions, in particular **WebSharper.JointJs** and **WebSharper.Rappid** - bringing awesome diagramming solutions to WebSharper:
 
   <div style="width:100%;min-height:400px;position:relative"><iframe style="position:absolute;border:none;width:100%;height:100%" src="http://try.websharper.com/embed/qwe2/00005D"></iframe></div>
 
 * Updating [CloudSharper](http://cloudsharper.com) with the latest WebSharper - this has been left on the backburner for several releases, now it's time to sync the two again.  Once this work is finished, CloudSharper will be your one-stop shop for online web and mobile development with C# and F#; quick data access, analytics and visualization, and a host of other interactive capabilities.
 
This article gave you a quick overview of WebSharper in 2015, and is by no means complete.  One thing is certain: WebSharper remains the primary choice for F# web developers, giving unparalleled productivity and access to the largest F# web ecosystem, and works well with a number of other efforts (ASP.NET, MVC, OWIN, Suave, Hopac, etc.) in the web and development space.

Happy coding and Happy Holidays!
