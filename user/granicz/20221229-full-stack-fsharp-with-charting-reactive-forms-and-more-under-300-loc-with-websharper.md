---
title: "Full-stack F# with charting, reactive forms, and more under 300 LOC with WebSharper"
categories: "f#,websharper,fsadvent"
abstract: ""
identity: "-1,-1"
---

This post is part of F# Advent 2022, a huge thanks to Sergey Tihon for organizing! If you are burning to read more, here is a list of my other F# Advent articles:

* 2021 - [Content-managed WebSharper apps with custom web controls and templating](https://intellifactory.com/user/granicz/20211230-ContentManagedWebSharperAppsWithCustomWebControlsAndTemplating)

* 2020 - [Variations for a WebSharper shopping cart](https://intellifactory.com/user/granicz/20201231-variations-for-a-websharper-shopping-cart)

* 2019 - [F# metablogging: introducing BlogEngine for your static markdown-based F# blog](https://intellifactory.com/user/granicz/20191226-f-metablogging-introducing-blogengine-for-your-static-markdown-based-f-blog)

* 2018 - [From enterprise to next-generation web: celebrating 11 years with WebSharper](https://intellifactory.com/user/granicz/20181210-from-enterprise-to-next-generation-web-celebrating-11-years-with-websharper)

* 2017 - [Serving SPAs](https://intellifactory.com/user/granicz/20171229-serving-spas)

* 2016 - [Simple reactive scenarios with WebSharper](https://intellifactory.com/user/granicz/20161231-simple-reactive-scenarios-with-websharper)

* 2015 - [WebSharper - a year in review](https://intellifactory.com/user/granicz/20151226-websharper-a-year-in-review)


---

WebSharper has always been about making F# web programming fun and super-productive. It encourages you to think outside the box of seasonal JavaScript libraries and gives you a fundamentally better, more robust and type-safe F# (and C#) toolset to get things done.

In this post, I will illustrate a handful of the many ways WebSharper will make you a better F#/C#/.NET full-stack web developer. A common theme as we go along will be externalizing the presentation layer and **putting your design into HTML template files**, and not into your source code. This avoids unnecessary recompilation and makes your design iterations much-much shorter. It's a tough habit to break if you are used to inline HTML, but once you master templating with WebSharper, there is no going back.

In the full-stack application we are building, we want to accomplish the following:

1. Charting some numbers and producing some pretty diagrams - for this, we'll be using [WebSharper.Charting](https://github.com/dotnet-websharper/charting), a library akin to FSharp.Charting but for WebSharper apps.

2. Collecting user input via nicely designed web forms - for this, we'll be using [WebSharper.Forms](https://github.com/dotnet-websharper/forms), a declarative, reactive forms library that adds type-safety to HTML input handling.

3. Calling the server for data and displaying it, and using hydration with server-side rendering to be more SEO and user-friendly.

---

> ### Try it yourself!
>
> 1. Clone the repo: `git clone https://github.com/granicz/FsAdvent2022`
>
> 2. cd into the project: `cd FsAdvent2022`
>
> 3. Build and run it: `dotnet run`
>
> 4. Navigate to `https://localhost:xxxx`, where `xxxx` is the port the app is started on
---

## The UI

First, we need to find a reasonable HTML+CSS template that suits our needs. It's a definite plus if you are a web designer or have a team member who can help. My advice as an F# dev: let them work for you so you can concentrate on your F# code instead.

After a quick look online, I decided to use [David Grzyb](https://github.com/davidgrzyb)'s [tailwind-admin-template](https://github.com/davidgrzyb/tailwind-admin-template) (thanks David!), because it fits my goal of demonstrating a couple things perfectly, and it's free to use. In either case, if you do decide to use his and his contributor's work, please consider helping them in any way you can.

After cloning the repo and studying the general structure of the HTML files, I decided to make a master template out of one of the source files (`index.html`), tidying up some content, and hooking up [WebSharper.UI](https://github.com/dotnet-websharper/ui) templating in it to make the left-hand side (LHS) menu bar, the hamburger menu, and the main content panel pluggable. Then I placed this updated `index.html` in a new WebSharper client-server app that I created with "`dotnet new websharper-web -lang f# -n MyApp`" (if you don't have the WebSharper templates installed, run "`dotnet new -i WebSharper.Templates`" first), overwriting `main.html` coming from the project template. There are still small issues left with the template file, such as flashing font-awesome icons on page refreshes and some occasional showing of the avatar dropdown menu as the page loads, so be aware. I also just added the template as is, without any preprocessing step, so no SCSS or CSS minification, but in a real-life project you will most likey want to set these up in your build.

Next, I removed `Client.fs` and `Remoting.fs` from the project, and wired up three pages (Home, Charts, and Forms) in `Site.fs`, which now looks like this:

```fsharp
namespace MyApp

open WebSharper
open WebSharper.Sitelets
open WebSharper.UI
open WebSharper.UI.Server

type EndPoint =
    | [<EndPoint "/">] Home
    | [<EndPoint "/charts">] Charts
    | [<EndPoint "/forms">] Forms

module Templating =
    let MenuBar (ctx: Context<EndPoint>) endpoint isMainMenuBar =
        let ( => ) (txt: string, icon: string) act =
            Templates.MainTemplate.MenuItem()
                .Title(txt)
                .ExtraCSSClasses(if endpoint = act then "active-nav-link" else "opacity-75 hover:opacity-100")
                .IconFaClass(icon)
                .TargetUrl(ctx.Link act)
                .PY(string (if isMainMenuBar then 4 else 2))
                .PL(string (if isMainMenuBar then 6 else 4))
                .Doc()
        [
            ("Home", "calendar") => EndPoint.Home
            ("Charts", "tachometer-alt") => EndPoint.Charts
            ("Forms", "align-left") => EndPoint.Forms
        ]

    /// Returns an HTML page based on the master template, given the endpoint to it, its title and body content.
    let Main ctx action (title: string) (body: Doc list) =
        Content.Page(
            Templates.MainTemplate()
                .NewPage(fun e -> JavaScript.JS.Alert "Add a nice popup here...")
                .Title(title)
                .MenuBar(MenuBar ctx action true)
                .MenuBarHamburger(MenuBar ctx action false)
                .Body(body)
                .Doc()
        )

module Site =
    open type WebSharper.UI.ClientServer

    let pageTitle (title: string) =
        Templates.MainTemplate.BodyHeader()
            .Title(title)
            .Doc()

    let HomePage ctx =
        Templating.Main ctx EndPoint.Home "Home" [
            pageTitle "Home"
        ]

    let ChartsPage ctx =
        Templating.Main ctx EndPoint.Charts "Charts" [
            pageTitle "Charts"
        ]

    let FormsPage ctx =
        Templating.Main ctx EndPoint.Forms "Forms" [
            pageTitle "Forms"
        ]

    [<Website>]
    let Main =
        Application.MultiPage (fun ctx endpoint ->
            match endpoint with
            | EndPoint.Home -> HomePage ctx
            | EndPoint.Charts -> ChartsPage ctx
            | EndPoint.Forms -> FormsPage ctx
        )
```

Note the absence of HTML combinators in the code and the use of templates instead - these are sourced from the master template I created earlier (which I omitted here, but you can find it in the project repository.) Clearly, before this file, we need to have the main template declared, for which we use the templating type provider:

```fsharp
[<JavaScript>]
module Templates =
    type MainTemplate = Templating.Template<"Main.html", ClientLoad.FromDocument, ServerLoad.WhenChanged>
```

When I run the app, I get a three-page empty site looking pretty good:

[![](https://i.imgur.com/d2Ez4Irh.png)](https://i.imgur.com/d2Ez4Ir.png)

---

## The charts page

I feel lazy so I head over to Try WebSharper to find some suitable chart candidates to show. I could use charts produced by a variety of JavaScript libraries, but I decide to keep it abstract and use [WebSharper.Charting](https://github.com/dotnet-websharper/charting), pick a combo radar chart [snippet](https://try.websharper.com/snippet/adam.granicz/00004H), and copy-paste it into a new file called `charts.fs`:

```fsharp
// Snippet from https://try.websharper.com/snippet/adam.granicz/00004H
module Charts

open WebSharper
open WebSharper.Charting
open WebSharper.UI

[<JavaScript>]
let Chart01 () =
    let labels =
        [| "Eating"; "Drinking"; "Sleeping";
           "Designing"; "Coding"; "Cycling"; "Running" |]
    let dataset1 = [|28.0; 48.0; 40.0; 19.0; 96.0; 27.0; 100.0|]
    let dataset2 = [|65.0; 59.0; 90.0; 81.0; 56.0; 55.0; 40.0|]
    
    let chart =
        Chart.Combine [
            Chart.Radar(Array.zip labels dataset1)
                .WithTitle("Day 1")
                .WithFillColor(Color.Rgba(151, 187, 205, 0.2))
                .WithStrokeColor(Color.Name "blue")
                .WithPointColor(Color.Name "darkblue")

            Chart.Radar(Array.zip labels dataset2)
                .WithTitle("Day 2")
                .WithFillColor(Color.Rgba(220, 220, 220, 0.2))
                .WithStrokeColor(Color.Name "green")
                .WithPointColor(Color.Name "darkgreen")
        ]
    
    Renderers.ChartJs.Render(chart, Size = Size(400, 300))
```

I see some red squiggles around `WebSharper.Charting`, `Chart.Combine`, `Renderers.ChartJs`, so I add a package reference to `WebSharper.Charting` and `WebSharper.ChartJs`, with the same `major.minor` version that WebSharper has in the project (`6.1`.xxx at the time of writing).

This resolves the compiler errors, and just in case, I remove the trailing "`.RunById "main"`" from the snippet, an artifact of coming from Try WebSharper.

Just for fun, I make another chart that plots `sin(x)/2x`, which within a 10 PI domain and a base reference point at `x=0` looks like:

```fsharp
open System

[<JavaScript>]
let Chart02 () =
    let chart =
        [-5.*Math.PI .. 0.1 .. 5.*Math.PI]
        |> Seq.map (fun x -> (if Math.Abs(x-0.1)<0.1 then "0" else ""), Math.Sin(x)/2./x)
        |> Chart.Line
    Renderers.ChartJs.Render(chart, Size = Size(400, 300))
```

Then I modify `ChartsPage` in `Site.fs` as follows:

```fsharp
let ChartsPage ctx =
    Templating.Main ctx EndPoint.Charts "Charts" [
        pageTitle "Charts"
        Templates.MainTemplate.Grid()
            .Columns(string 2)
            .GridCards([
                Templates.MainTemplate.GridCard()
                    .Title("Activities")
                    .Content(client (Charts.Chart01()))
                    .Doc()
                Templates.MainTemplate.GridCard()
                    .Title("Heartbeat")
                    .Content(client (Charts.Chart02()))
                    .Doc()
            ])
            .Doc()
    ]
```

Note the lack of HTML functions again, and the use of local templates `Grid` and `GridCard`, which I created earlier in `Main.html` based on the original HTML files.

At this point, our work is done: the page renders pretty sweet so we are ready to move on to the next page.

[![](https://i.imgur.com/ozgeWTWh.png)](https://i.imgur.com/ozgeWTW.png)

---

## The forms page

The WebSharper.UI.Client namespace has a large collection of declarative input controls, such as `Doc.InputType.Text`,`*.CheckBox`, `*.Email`, `*.Date`, `*.File`, `*.Float`, `*.Int`, `*.TextArea` and others, that you can use in your F# code. These are also available to bind from templates, which is exactly what we need in order to rid the F# code from UI artifacts. And this is where we bring in [WebSharper.Forms](https://github.com/dotnet-websharper/forms) - WebSharper's reactive forms library with awesome features.

Two-way binding an input control requires a simple `ws-var="xxx"` attribute on the `<input>` tag in your HTML template. For this example, I will use the "Customer information" section of `forms.html` of the template we downloaded. I grab this section from there and place it inside `Main.html` in our project, marking it up as a new inner template and sprinkling in the `ws-var` attributes on the input tags.

Then I start a new `Forms.fs` file and add the WebSharper.Forms boilerplate to define an **abstract form** with a number of input controls yielding values (name, email, street, etc. - all initialized to an empty string), a submitter that will trigger validation (which we could use but don't, and instead treat it with HTML in the template), and a dummy run and render functions - and this time, we prepare ourselves for server-side pre-rendering and hydration too:

```fsharp
...
module Forms =
    open WebSharper.Forms

    let CustomerInformation () =
        if IsClient then
            Form.Return (fun name email street city country zip cc ->
                name, email, street, city, country, zip, cc)
            <*> Form.Yield ""
            <*> Form.Yield ""
            <*> Form.Yield ""
            <*> Form.Yield ""
            <*> Form.Yield ""
            <*> Form.Yield ""
            <*> Form.Yield ""
            |> Form.WithSubmit
            |> Form.Run (fun ((name, email, street, city, country, zip, cc) as data) ->
                () // Your submit logic here
            )
            |> Form.Render (fun name email street city country zip cc submitter ->
                Doc.Empty // Rendering the form on the client
            )
        else
            Doc.Empty // Rendering the form on the server
```

The power of WebSharper.Forms comes from being able to separate the abstract, declarative UI specification (a sequence of `Form.Yield*` calls composed with `<*>`, and a final `Form.Return` to bundle each inner value together) from its rendering, and transfering/using the data extracted on successful validation (again, which we omit here and handle in the template directly) via `Form.Run`.

Let's deal with `Form.Run` first. This would be the place for sending data back to the server, persisting it, and coming back with a result to show the user. For our purposes here, we just pop up an alert box with the collected (strongly typed) data:

```fsharp
...
|> Form.Run (fun ((name, email, street, city, country, zip, cc) as data) ->
    JS.Alert $"You submitted: {data}"
)
```

Rendering will use the inner template we built, and each of the `ws-var="..."` attributes we added to it will expose a member that allows us to "connect" the input control and a reactive variable - which, to no surprise, are supplied by the lambda parameters, derived from the preceding field composition:

```fsharp
    ...
    |> Form.Render (fun name email street city country zip cc submitter ->
        Templates.MainTemplate.CheckoutForm()
            .Name(name)
            .Email(email)
            .Street(street)
            .City(city)
            .Country(country)
            .Zip(zip)
            .CreditCard(cc)
            .Submit(fun e -> submitter.Trigger())
            .Doc()
    )
else
    Templates.MainTemplate.CheckoutForm()
        .Name("")
        .Email("")
        .Street("")
        .City("")
        .Country("")
        .Zip("")
        .CreditCard("")
        .Doc()
```

The `else` branch belongs to the `IsClient` check and it provides the UI we can compute on the server. This makes hydration super-easy and the last piece is wiring that up (note the call to `hydrate`) inside `FormsPage` in `Site.fs`:

```fsharp
let FormsPage ctx =
    Templating.Main ctx EndPoint.Forms "Forms" [
        pageTitle "Forms"
        Templates.MainTemplate.Grid()
            .Columns(string 2)
            .GridCards([
                Templates.MainTemplate.GridCard()
                    .Content(hydrate (Forms.CustomerInformation()))
                    .Doc()
            ])
            .Doc()
    ]
```

The end result couldn't look better:

[![](https://i.imgur.com/FRL9OoQh.png)](https://i.imgur.com/FRL9OoQ.png)

If you keep refreshing this page, you will see that the form comes statically rendered - as it should because we used `hydrate`. Feel free to switch that to using `client` instead to see the difference - you should get a visual flicker on each page refresh, due to the form being dynamically constructed after page initialization. Easy, seamless hydration for the win!

---

## Where are we at?

Let's summarize what we learned so far in terms of full-stack development with WebSharper:

* We write client-side code in F#, and mark each module or function we want to run on the client with the `[<JavaScript>]` attribute, and WebSharper takes care of compiling these to JavaScript and including the generated script in the containing page. If your client-side code uses external JavaScript libraries, those will be automatically included in every page that needs them as well. You don't have to include anything manually other than what your template requires.

* We write the server-side in F#, and use the `client` and `hydrate` functions (from `WebSharper.UI.ClientServer`) to embed client-side content in server-generated pages. If your client-side code has free variables, ie. it uses values defined in server-side territory, those will be serialized automatically and shipped to the client at page initialization to run as intended. Hydration is super-easy.

* You can freely mix server-side and client-side code, place them in a single file or break them out to separate files by application funcionality as we did in this article. The choice is yours. You don't need to place server vs client-side code in different projects, or create even more projects to share code in between tiers. It works without any burden.

* By using HTML templates instead of inline HTML, we can make instant design changes without recompiling our project. Need to completely reorganize a data grid or rework a web form? No problem, just change your HTML template and reload the app in your browser. Check the optional arguments on your templating type provider call, these control the loading behavior.

---

## The home page

Now, for our final page, the home page, let's turn to client-server communication. What we are after is a table/grid showing some data fetched from the server, as shown below. The twist this time is dealing with the delay inherent in this sort of communication and how the UI can smooth things out for us - but I will leave that unhandled for now.

[![](https://i.imgur.com/1vWWnU8h.png)](https://i.imgur.com/1vWWnU8.png)

As before, I create a new file called `Home.fs` and place the following code into it:

```fsharp
namespace MyApp

open WebSharper

module Home =
    [<JavaScript>]
    type UserDTO =
        {
            FirstName: string
            LastName: string
            PhoneNumber: string
            Email: string
        }

    module Server =
        /// Generates some random user DTOs
        [<Rpc>]
        let FetchUsers() =
            let fetchUser() =
                let rnd = System.Random()
                let oneOf lst = List.item (rnd.Next(0, List.length lst)) lst
                let firstName = oneOf ["John"; "Paul"; "Steven"; "Joseph"; "Ariel"; "Eve"; "Margaret"]
                let lastName = oneOf ["Smith"; "Reeds"; "Wolfram"; "Heureka"; "Burns"; "Weston"; "Price"]
                let emailDomain = oneOf ["gmail.com"; "outlook.com"; "yahoo.com"; "hotmail.com"]
                {
                    FirstName = firstName
                    LastName = lastName
                    PhoneNumber = $"{rnd.Next(100, 1000)}-{rnd.Next(100, 1000)}-{rnd.Next(1000, 10000)}"
                    Email = $"{firstName.ToLower()}.{lastName.ToLower()}.{rnd.Next(10, 100)}@{emailDomain}"
                }
            async {
                let users = [for i in 1 .. 9 -> fetchUser()]
                return users
            }

    [<JavaScript>]
    module Client =
        open WebSharper.UI
        open WebSharper.UI.Client

        let UserTable() =
            let renderUserRow (user: UserDTO) =
                Templates.MainTemplate.UserRow()
                    .FirstName(user.FirstName)
                    .LastName(user.LastName)
                    .PhoneNumber(user.PhoneNumber)
                    .Email(user.Email)
            if IsClient then
                Templates.MainTemplate.UserTable()
                    .Rows(
                        async {
                            let! users = Server.FetchUsers()
                            return
                                users
                                |> List.map (fun user ->
                                    (renderUserRow user)
                                        .Doc()
                                )
                                |> Doc.Concat
                        }
                        |> Doc.Async
                    )
                    .Doc()
            else
                Templates.MainTemplate.UserTable()
                    .Rows(
                        [ for i in 1 .. 9 ->
                            { FirstName="John"; LastName="Smith"; PhoneNumber="123-456-7890"; Email="john.smith@gmail.com" }]
                        |> List.map (fun user ->
                            (renderUserRow user)
                                .ExtraCSS("filter blur-sm")
                                .Doc()
                        )
                        |> Doc.Concat
                    )
                    .Doc()
```

This defines a DTO type we use to communicate between the client and the server, and an RPC function that computes some users in this DTO format asynchronously.

The `Client` module contains the table rendering, using our `UserTable` inner template coming from `Main.html`. The code is somewhat repetitive: first, we statically generate dummy rows on the server, applying extra CSS classes on each row to blur them. This then after a split second is replaced with the client-side version after page initialization, lighting up the user table with actual data.

The main pattern in this client-server communication is the following:

```fsharp
...
async {
    let! users = Server.FetchUsers()
    return <<...a Doc value representing our users...>>
}
|> Doc.Async
```

Here, `Doc.Async` is doing the heavy-lifting: it immediately yields an empty Doc value, then evaluates the async block to obtain a new Doc value, and it replaces the empty Doc value with it in the DOM. This also means that if your RPC function takes a while to return, or if you use web requests to third-party hosts that end up taking a while, you will be seeing an empty table for that duration before the table finally populates.

There are a few ways of dealing with that and showing a waiting animation such as a pulsating/blurred table - but I will leave that for a future article.

---

## Exploring further

To make things even snappier, you could turn the above sitelet into an SPA, doing the routing between the pages on the client, and updating the LHS menubar and the main content section accordingly. You can read up on how to accomplish this in my [Serving SPAs](https://intellifactory.com/user/granicz/20171229-serving-spas) article.

Whether you make an SPA out of it or just keep it as a self-serving sitelet, you can easily deploy your app into Azure or the web host of your choice by copying the output folder. You can also switch the RPC/server-side into a microservice and deploy the UI and the backend separately. In this case, you should enable CORS and change the RPC call to an HTTP request.

---

## News: the upcoming WebSharper 7

This post wouldn't be complete without a couple WebSharper announcements. First off, WebSharper turned 15 this year - a huge thanks to our users and contributors for making it better and better! Each major version we released feels like a new project, and WebSharper 6 is still less than a year old.

Back in 2018, I wrote an article [for the 11th year mark](https://intellifactory.com/user/granicz/20181210-from-enterprise-to-next-generation-web-celebrating-11-years-with-websharper), which outlines a lot of the direction we have been pursuing ever since - opening up a WASM-based execution with [Bolero](https://fsbolero.io) and building further on WASM AOT. That article also walks through an app that you can cross-compile to JavaScript and run on WASM at the same time, requiring minimal code changes - given that WebSharper and Bolero share the same HTML notation and templating capabilities.

To further align WebSharper and Bolero, we will be rolling out a new, experimental DOM/HTML construction library using the (more efficient) computation expression (CE)-based syntax Bolero switched to earlier. This is a minor enhancement and will be available in addition to the standard WebSharper.UI reactive representation.

A more significant update is coming to the JavaScript output that WebSharper generates - going from the "works very efficiently but don't look at it" to modern, ESM-based modular output, with extra attention on preserving .NET semantics in the generated code. This opens up the possibility of deep integration with all modern JavaScript libraries, including but not limited to Lit, React, Vue, Svelte, SolidJS, etc., working with custom web controls and shadow DOM, and integrating with TypeScript.

Up to WebSharper 6, for 8+ years now, you have had the choice of using WebSharper.UI's reactive dataflow capabilities (FRP with signals, etc.) and composite `ListModel`'s to generate reactive UIs that are rendered only once, with subsequent model changes propagating to the UI with minimal DOM updates, and without any virtual DOM or DOM diffing. You also have [WebSharper.Mvu](https://github.com/dotnet-websharper/mvu) that builds on a custom implementation of The Elm Architecture (TEA) to give you a modified MVU pattern with the benefits of arbitrarily complex data models, using lenses to describe inner updates. This also uses dynamic dataflow instead of DOM diffing, but comes at the notational cost of lenses and/or the V shorthand.

With WebSharper 7, we will be extending your UI choices based on MVU, built on [Elmish](https://github.com/elmish/elmish) instead of a custom TEA layer. I am really excited about opening up the Elmish ecosystem to WebSharper, the first piece of which you can find in [this PR](https://github.com/elmish/elmish/pull/264).

Again, using signals and Svelte/SolidJS-like dataflow remains available with WebSharper.UI.

If you have any questions or would like to give us a hand, don't hesitate to reach me on [F# Web Discord](https://discord.com/channels/196693847965696000/386379055827779595) or the [F# Web Slack channel](https://app.slack.com/client/T04BJKUMU/C10DLSHQV), or file a ticket in GitHub, or on the [WebSharper Forums](https://forums.websharper.com/topics/all).

Happy coding and Happy 2023!
