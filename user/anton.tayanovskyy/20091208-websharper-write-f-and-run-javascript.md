---
title: "WebSharper - Write F# and Run JavaScript"
categories: "f#,websharper,euler"
abstract: ""
identity: "1457,74667"
---
It is an exciting time to be working for IntelliFactory. After quite a bit of hard work, we finally have a public beta release of the [WebSharper platform](https://websharper.com)[^1]. As we continue to work on it, I encourage you to download and play with the current beta release.

WebSharper aims to change the way you think about web programming, no more and no less. The idea behind it is very simple. Instead of HTML + JavaScript + PHP/C#/Java code, you write F#, and let the compiler do its magic to get a working AJAX website. Alternatively, you develop a small component and expose it as an ASP.NET control, without having to rewrite your website from scratch.

Let me start with a simple, indeed primitive, example. Let us take the the first problem from [Project Euler](http://projecteuler.net/) (a great source of profitable amusement for many of us), solve it in F#, and run it in the browser. To play with it, download and install the beta release of the [WebSharper platform](https://websharper.com)[^2], start a new WebSharper solution, and change the project code to the following:

```fsharp
namespace WebSharperProject

open IntelliFactory.WebSharper
open IntelliFactory.WebSharper.Html 

[<assembly: WebSharperAssembly>]
do ()

[<JavaScriptType>]
module Math = 

    /// Find the sum of all the multiples of 3 or 5 below 1000.
    [<JavaScript>]
    let ProjectEuler0001 =
        seq { 0 .. 999 }
        |> Seq.filter (fun x -> x % 3 = 0 || x % 5 = 0)
        |> Seq.sum

[<JavaScriptType>]
type Example() = 
    inherit Web.Control()

    [<JavaScript>]
    override this.Body = 
        let answer = Span []
        Div [
            Div [
                Span ["The answer is: "]
                answer
            ]
            Input [Type "Button"; Value "Calculate"]
            |> On Events.Click (fun _ ->
                answer.Text <- string Math.ProjectEuler0001)
        ]
```

Then change `Default.aspx` to reference the newly defined `Example` control:

```xml
  <%@ Page Language="C#" %>
  <html xmlns="http://www.w3.org/1999/xhtml">
    <head runat="server">
      <WebSharper:ScriptManager runat="server" />
    </head>
    <body>
      <ws:Example runat="server" />
    </body>
  </html>
```

This is a small example, but there is a lot to get excited about already. Take powerful F# type-checking, functional abstractions, embedded HTML combinators, or ASP.NET integration...

Yes, Project Euler problems are not exactly representative of the tasks a typical web programmer faces. But I firmly believe that a platform that makes difficult things possible should make simlpe things simple.

I will be blogging with more posts on WebSharper in the coming days. In the meanwhile, the impatient should definitely check out the [Demos](https://try.websharper.com)[^3].


[^1]: This link has been updated to point to the WebSharper home page.

[^2]: This link has been updated to point to the WebSharper home page.

[^3]: This link has been updated to point to the Try WebSharper home page.
