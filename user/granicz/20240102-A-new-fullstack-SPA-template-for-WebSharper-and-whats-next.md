---
title: "A new fullstack SPA template for WebSharper and what's next"
categories: "f#,websharper,fullstack,template,tailwind,vite,fsadvent"
abstract: ""
identity: "-1,-1"
---

First off, Happy 2024 everyone! This is the last (belated, sorry!) entry in the F# Advent (FSA) 2023 calendar, and I would like to thank Sergey for organizing, and all those who contributed to make it AGAIN a goldmine for F# content! No better way to spend December and the start of the new year than reading about how productive F# can make you!


## Introducing `fullstack-spa`

| Tables | Charts | Forms |
|--|--|--|
| [![](/assets/fullstack-spa-Tables.png)](/assets/fullstack-spa-Tables.png) | [![](/assets/fullstack-spa-Charting.png)](/assets/fullstack-spa-Charting.png) | [![](/assets/fullstack-spa-Forms.png)](/assets/fullstack-spa-Forms.png) |

> Currently, WebSharper 7 is in beta, latest version is `beta-4` - available from the WebSharper developer feed. Check [here](https://docs.websharper.com/basics/nuget/) about configuring your NuGet/Paket client to use these releases, you won't be able to compile the [project](https://github.com/granicz/fullstack-spa) otherwise!
> ### Try it!
>
> 1. Clone the repo: `git clone https://github.com/granicz/fullstack-spa`
>
> 2. cd into the project: `cd fullstack-spa`
>
> 3. Build and run it: `dotnet run`
>
> 4. Navigate to `https://localhost:xxxx`, where `xxxx` is the port the app is started on (as reported by vite)

---

Recently, I teamed up with others to build a handful of new project templates (PTs) for the upcoming WebSharper 7, and to release these individually (so you can install them on-demand) as we go without any special ties to [WebSharper.Templates](https://www.nuget.org/packages/WebSharper.Templates/), the main collection of PTs that contains the standard fullstack (`websharper-web`), SPA (`websharper-spa`), static website (`websharper-html`), library (`websharper-lib`), microservice (`websharper-svc`), .NET proxy (`websharper-prx`), and JavaScript binding (`websharper-ext`) PTs that you might already be familiar with.

For one of these new PTs, I decided to take my "Full-stack F# with charting, reactive forms, and more under 300 LOC with WebSharper" [FSAdvent2022 project](https://github.com/granicz/fsadvent2022) from last year's FSA, and modernize it with a few overdue changes and enhancements.

[<img src="/assets/X-fullstack-300.png" alt="image" width="50%" height="auto" />](https://twitter.com/granicz/status/1609060870289489920)

For the general idea and walkthrough of the charting and forms bits, feel free to refer to the [article](https://intellifactory.com/user/granicz/20221229-full-stack-fsharp-with-charting-reactive-forms-and-more-under-300-loc-with-websharper) - it's a fun read! As for the rest of the updated version, here is a quick summary.

### Tailwind for UI

The `fullstack-spa` PT uses an MIT-licensed [Tailwind](https://tailwindcss.com/)-based template, [Windmill Dashboard](https://github.com/estevanmaito/windmill-dashboard) by [@estevanmaito](https://github.com/estevanmaito), part of the [Windmill UI suite](https://windmillui.com/). It's a lovely free resource for web devs, supports dark/light modes, comes with a bundle of prebuilt admin and other components/pages, etc. - so be sure to support Estevan's work in any way you can.

Changing, or even swapping out the presentation layer entirely only requires changing a single file (`client/index.html`). Since the PT uses WebSharper.UI templating, be sure to correctly apply the templating attributes (`ws-*`) for any new UI building block. These then will expose new types and members for your F# code to use for each template, placeholder, etc in the source template. Type-safe templating for the win!

### Code reorganization and a separate build project

Instead of the [FSAdvent2022 application](https://github.com/granicz/fsadvent2022)'s single F# project to hold both the server and client-side, these are now split into three separate F# projects: `client`, `server`, and `shared`. This is not strictly necessary as the WebSharper Booster (which is responsible for cached compilation of WebSharper projects) usually recompiles colocated client+server code in a 2-3-4 seconds, it can, however, result in faster turn-around times if your changes to be compiled are only in your client-side code.

The new build project (`Build.fsproj` and `build/build.fs`) follows the typical FAKE layout that you might have seen with other web apps, with a `Run` target to build and watch the client/server projects and starting a `vite` server. The client project uses the WebSharper CLI to provide a more streamlined watch functionality, bypassing ordinary MSBuild logic, and recompilation after changing+saving source files takes ~1.5 seconds. The page is also automatically refreshed, thanks to vite's development mode.

[<img src="/assets/watch-recompilation.png" alt="image" width="50%" height="auto">](/assets/watch-recompilation.png)


### ESM output and Vite

WebSharper 7 produces one-file-per-type (OFPT) ESM output and TypeScript declarations, and while it has its own dead-code eliminator and bundler, it is also designed to readily integrate with standard JavaScript tooling, and this PT is no exception: it comes with [vite](https://vitejs.dev/) for running an ESM development server and providing production mode bundling.

To start your app, all you need is to trigger the `Run` target:

```
dotnet run
```

[<img src="/assets/vite.png" alt="image" width="50%" height="auto">](/assets/vite.png)


### Femto

WebSharper 7 JavaScript bindings/NuGet packages are compatible with [femto](https://github.com/Zaid-Ajaj/Femto) for managing their underlying npm dependencies. You can use femto to finetune/update/validate package references in your `packages.config` as needed, and otherwise just continue adding new WebSharper references as NuGet.


### Record-style RPCs

WebSharper 7 remoting supports record-style RPCs, in addition to annotated RPCs and classes used in previous releases. One added benefit is that WebSharper RPCs are now also callable from [Fable.Remoting](https://github.com/Zaid-Ajaj/Fable.Remoting) clients (as long as the types involved are supported by the caller code), and vice versa with minimal code changes.

For instance, here is the RPC contract from `shared` that can supply an array of customer transactions:

```fsharp
namespace Shared

open WebSharper

module DTO =
    type Client =
        {
            FirstName: string
            LastName: string
            Position: string
            AvatarUrl: string
        }

    type Status =
        | Approved
        | Pending
        | Denied
        | Expired

    type Transaction =
        {
            To: Client
            Amount: float
            Status: Status
            Date: System.DateTime
        }

(*
 * A record type to hold our RPCs.
 * For discovering these RPCs, they need to be marked
 * with the [<Rpc>] attribute.
 *)
type IApi = {
    [<Rpc>]
    GetAllTransactions : unit -> Async<DTO.Transaction array>
}
```

And here is the implementation code from `server`:

```fsharp
module Service

open Shared
open Shared.DTO

let server : IApi = {
    GetAllTransactions = fun () ->
        async {
            ...
        }
    }

// Register the server-side handler for the RPCs.
WebSharper.Core.Remoting.AddHandler typeof<IApi> server
```

## What's next?

If you haven't tried it yet, the above PT is a good way to give WebSharper 7 beta a go, and share your feedback on the [WebSharper forums](https://forums.websharper.com), or in the [core GitHub repo](https://github.com/dotnet-websharper/core) - your bug reports, PRs, questions/comments are welcome!

Keep in mind that the final WebSharper 7 might undergo API and other changes in the coming weeks. We expect to ship WebSharper 7 around/by mid-Q1 2024, followed with a comprehensive update to the documentation and main websites.


### Proxies, proxies

More PTs are underway. Some of these PTs will showcase using Elmish+React+JSX and various UI libraries, and help you jumpstart your journey with WebSharper.

The standard UI paradigm for WebSharper applications has been a native-F#, dynamic dataflow-based reactive model via [WebSharper.UI](https://github.com/dotnet-websharper/ui), similar to the increasingly popular SolidJS/Svelte approach in JS applications. In addition, an Elmish-like Model-View-Update (MVU) library has been implemented on top as [WebSharper.Mvu](https://github.com/dotnet-websharper/mvu), and later, an actual Elmish implementation was added under [WebSharper.Elmish](https://github.com/elmish/elmish). This latter library used a [proxy](https://developers.websharper.com/docs/v4.x/fs/proxying), WebSharper's way of providing translation over arbitrary .NET APIs, to bring Elmish to WebSharper.

Recently, we have been investigating adding other proxies around some of the notable non-WebSharper libraries from the F# web ecosystem (Feliz, etc.) and Fable itself. This will soon open up interesting possibilities for both Fable and WebSharper developers, enable Fable web developers to benchmark code generation using WebSharper on their Fable apps, and to use WebSharper features otherwise not available in their stacks. For instance, essentially, you can soon take a Fable SPA and compile it with WebSharper without requiring any code changes, enhance it with WebSharper features/code, and host it in a WebSharper sitelet.

If you would like to further explore what WebSharper can do for your existing .NET web apps, feel free to DM me on X/FB/LinkedIn, or use our [contact form](https://intellifactory.com/contact) to get in touch.


### I am a C# developer, can I use WebSharper?

Absolutely! WebSharper supports compiling both F# and C#, and even though the C# compilation path is not as feature rich as the F# one, it has been successfully used in various .NET/C# web projects. If you would like to help volunteer adding more C# features or fix/enhance the features already implemented, don't hesitate to reach out.


### What's left for WebSharper 7?

One of the many unique WebSharper features is the ability to embed client-side code in server-side markup, giving you a true fullstack developer experience. The SPA PT from above and other F# SPA projects you might have encountered before (SAFE or other Fable-based variants) are primarily client-centric, and approach your app as an exercise to enhance an HTML page with JavaScript code generated from an F# codebase. Any server-side code is then only loosely coupled, and the remoting layer makes no guarantees about matching the contract required by both tiers.

For WebSharper apps, there is another mode of operation, which has traditionally been the default mode: using [sitelets](https://developers.websharper.com/docs/v4.x/fs/sitelets).

Sitelets are self-hosted ASP.NET Core applications, but can also be made to run in Giraffe or Saturn using an [experimental bridge](https://github.com/intellifactory/sitelets), although recently, this received less focus than it deserved.

Sitelets bring client and server code under a unified programming model, give you isomorphic HTML over a range of UI paradigms (including the default WebSharper.UI reactive model with type-safe templating), and enable you to define how server responses (that may or may not include dynamic, client-side content) are computed for incoming requests, which themselves are mapped to values of endpoint types.

```fsharp
type MyEndPoint =
    | [<EndPoint "/">] Home
    | [<EndPoint "/echo">] Echo of string
    ...

[Website]
let Main =
    Application.MultiPage (fun ctx -> function
        | Home ->
            Content.Page (...) // <--- Your home page content here
        | Echo msg ->
            Content.Text msg
    )
```

In the code above, `Content.Page` is a helper that constructs HTML responses, which takes an entire document or a document fragment (body), and will automatically include all required resources (npm and other) via a page-wide script generated at compile time. This is the piece currently in development before WebSharper 7 can be released: generating these include scripts for every sitelet page and making them available to post-compilation bundling.

It's likely that the final design won't arise until WebSharper 8, and WebSharper 7 will only include a basic implementation.

Just for fun and to give you an idea of the complexities involved, in the content passed to `Page.Content`, you can embed a variety of constructs that may in turn have dynamic, client-side aspects, among others:

1. Server-side HTML with `OnAfterRender` and other event handlers:

    ```fsharp
    div [] [text "Hello world!"]

    div [
        on.afterRender (fun e -> Console.Log "OAR!")
    ] [text "I am a div with OAR"]

    div [
        on.click (fun e args -> Console.Log "clicked!")
    ] [text "I am a div with a click handler"]
    ```

2. Client-side HTML, using `client` - these execute after the containing page loads:

    ```fsharp
    client (div [] [text "Hello world!"])
    ```

3. Hydrated client-side HTML, using `hydrate` - these render on the server, with any client-side functionality lit up after page load:

    ```fsharp
    hydrate (RenderClientForm())
    ```

4. Client-side controls:

    ```fsharp
    Doc.WebControl (MyControl("I am a client-side control"))
    ```

    where `MyControl` is defined as:

    ```fsharp
    type MyControl(a: string) =
        inherit Web.Control()

        [<JavaScript>]
        override this.Body =
            div [
                on.click (fun e arg -> Console.Log "MyControl clicked")
            ] [text "Click me, I am a client-side control"]
    ```

5. Template-driven components with event bindings:

    ```fsharp
    type TemplateFile = Template<
        @"wwwroot\template.html", ClientLoad.FromDocument, ServerLoad.WhenChanged>

    TemplateFile.MyComponent()
        .Title("Click me, I am a templated component")
        .OnClick(fun e ->
            Console.Log $"MyComponent onClick => Textbox has \"{e.Vars.Textbox.Value}\"")
        .Doc()
    ```


### What about support for F# 8?

Once WebSharper 7 rolls out, we expect to make a short iteration to bring in all the latest [F# 8](https://github.com/dotnet-websharper/core/issues/1366) and [C# 12](https://github.com/dotnet-websharper/core/issues/1367) features, and release WebSharper 8, putting us back on track with frequent incremental releases and matching the latest F# and .NET SDK version. Be sure to check on the [release bucket](https://github.com/orgs/dotnet-websharper/projects/1) to see what items are being worked on. Get involved on GitHub, your help is always greatly appreciated!

All in all, a great year ahead! I can't wait to see what the WebSharper community will accomplish in 2024!

Happy 2024 and happy coding!
