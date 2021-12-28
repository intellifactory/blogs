---
title: "Client-Server Routing with WebSharper"
categories: "f#,websharper,ui,routing,fsadvent"
abstract: ""
identity: "-2,-2"
---

This article is part of [F# Advent](https://sergeytihon.com/category/f-advent/). Thanks to [Sergey Tihon](https://twitter.com/sergey_tihon) for organizing!

The main advantage of using a single language for a full-stack project (be it in JavaScript/TypeScript, or better, F# :) ) is to have common abstractions shared between server and client-side code.
This makes making updates to full-stack project much safer and quicker. WebSharper's philosophy is to enable full-stack F#/C# projects that take advantage of this.

Today we are looking at how to share a sitemap and router (mapping between URLs and locations in the site) between client and server.
We will be able to:

  * Generate links from site locations (Endpoint values)
  * Handle certain link navigations (URL changes) on the client without reloading the page, while reloading from server when we enter another part of the site

So routing duties are split between the server and client, meanwhile both sides are able to link to any page on the site safely.
This is a compromise between a many-page site and a single-page application which can be useful where speed of navigation is preferred in some page navigations and full page reload for up-to date information or server-side processing in otheres.

Our demo project is a rudimentary blog engine, that has a Home page with the full list of articles, Article pages for each entry for reading, and an Edit page for editing a selected article or a creating a new one.
The sitemap is represented by this type:

```fsharp
    type EndPoint =
        | [<EndPoint "/">] Home
        | [<EndPoint "/article">] Article of id: int
        | [<EndPoint "/edit">] Edit of id: int
```
Navigation between Home and individual articles don't reload the page, while entering/exiting Edit pages do.

Splitting between these lines can be centralized with an active pattern:

```fsharp
    let (|ReadPage|EditPage|) e =
        match e with
        | Home
        | Article _ -> ReadPage
        | Edit _ -> EditPage
```

# Setup

As this sample uses some easy persistance by writing/reading `.txt` files in an `/articles` folder, there is no online demo.

To check out the sample project, run `git clone https://github.com/Jand42/MiniBlog` then `dotnet run` from the MiniBlog folder.
Then open `http://localhost:5000/`

![Screenshot of home page](https://i.imgur.com/UpwOWx2.png)

# How WebSharper creates routers

If you have and F# union EndPoint type as above, you can create a router instance with `let siteRouter = Router.Infer<EndPoint>()` that uses the union type's annotations.
We can reuse this to guide the both server-side and client-side parsing of URLs and link generation in a uniform way.

Let's look at first what the server's doing with it:

```fsharp
    [<Website>]
    let Main =
        Sitelet.New siteRouter (fun ctx endpoint ->
            match endpoint with
            | ReadPage -> 
                Templating.Main ctx endpoint "MiniBlog" [
                    div [] [client <@ Client.ReadPage endpoint @>]
                ]
            | EditPage -> 
                Templating.Main ctx endpoint "Edit article" [
                    div [] [client <@ Client.EditPage endpoint @>]
                ]
        )
```

If we encounter a read-only page, we set up the client by passing along our parsed endpoint value, same for an edit page.
The `client <@ ... @>` helper means: this is content to be generated in JavaScript, server just returns a placeholder and browser does the rest.
Arguments can be passed along and they are auto-serialized.

We also see we are creating a `Sitelet` object, which is essentially a router and a rendering function paired together in a type-safe abstraction.

# Setting up client-side routing

In client-side code (to be transpiled to JavaScript), this is where the magic happens:

```fsharp
    let ReadPage endpoint =
        let endpointVar = Var.Create endpoint
        
        siteRouter
        |> Router.Filter (
            function
            | Routing.ReadPage -> true
            | _ -> false
        )
        |> Router.InstallInto endpointVar Home
```

For the `ReadPage` handler, we only want to handle certain pages, so we filter for it, then install the router as an two-way binding between the URL and a reactive variable.

Then we can tie rendering our page by mapping from this `Var`, calling the server asynchronously for the content we need in simple records.

This is the part rendering the `Home` page, containing full list of entries, using the templating engine to define our exact visuals in Html:

```fsharp
        endpointVar.View
        |> Doc.BindView (
            function
            | Home ->
                async {
                    let! articles = Server.GetArticles()
                    return
                        articles 
                        |> Seq.map (fun a ->
                            Templates.MainTemplate.Article()
                                .Title(a.Title)
                                .Text(createContent a.Text)
                                .ReadArticleLink(siteRouter.Link (Article a.Id))
                                .EditArticleLink(siteRouter.Link (Edit a.Id))
                                .Doc()
                        )
                        |> Doc.Concat
                }
                |> Doc.Async
```

# Thank you for reading!

Feel free to discuss on [Gitter](https://gitter.im/intellifactory/websharper)

Happy New Year!