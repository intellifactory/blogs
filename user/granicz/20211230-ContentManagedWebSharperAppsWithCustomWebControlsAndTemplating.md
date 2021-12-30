---
title: "Content-managed WebSharper apps with custom web controls and templating"
categories: "f#,websharper,ui,templating,cms,fsadvent"
abstract: ""
identity: "-1,-1"
---

(This post is part of [F# Advent](https://sergeytihon.com/2021/10/18/f-advent-calendar-2021/), a huge thanks to [Sergey Tihon](https://github.com/sergey-tihon) for organizing!)

A while ago I gave a talk at [F# Exchange 2021](https://skillsmatter.com/conferences/13254-f-sharp-exchange-2021), where I discussed how WebSharper [sitelets](https://developers.websharper.com/docs/v4.x/fs/sitelets) can be extended with a CMS primitive that allows sitelet pages to be "content managed." This has some major implications on how you could design more maintainable content-centric web applications by enabling runtime access to their (designated) content.

In this article, I will discuss the details of making the underlying CMS web controls and give some insights on making the most of WebSharper web controls and UI templates.

> You can follow along and run the entire app via:
>
>    1. `git clone https://github.com/granicz/BlogWithCMS`
>    2. `cd MyBlog`
>    3. `dotnet run`

Below is what our final app looks like with the CMS functionality applied on a micro blogging app, showing the results in two tabs: viewing a given post in one and editing it in another:

![Our final app](/assets/BlogWithCMS-EditAndView.gif)

## Web controls

WebSharper web controls go way back to ASP.NET where they served as a bridge between client-side/dynamic content/behavior and ASP.NET server-side markup (you can read more on the [ASP.NET page](https://developers.websharper.com/docs/v4.x/fs/aspnet) of the documentation.) They do so by providing a `System.Web.Control` derivative type called `WebSharper.Web.Control`, which you can inherit from to create your own server-side controls.

The main thing to remember with web controls, though, is that they are merely a placeholder when rendered on the server, and they light up automatically on the client by evaluating their `Body` property (as indicated by the `[<JavaScript>]` annotation), and embedding the result into this placeholder. In other words, the content appears shortly after loading the page, causing a visual flicker.

Consider the following snippet to implement a custom button control:

```fsharp
open WebSharper
open WebSharper.JavaScript
open WebSharper.UI.Html

type MyButton() =
    inherit Web.Control()

    [<JavaScript>]
    override this.Body =
        button [
            on.click (fun _ _ -> JS.Alert "clicked!")
        ] [text "Hello world!"] :> _
```

To embed an instance of this web control into your UI markup, you can use `Doc.WebControl`:

```fsharp
div [] [Doc.WebControl <| MyButton()]
```

## Server-side rendering of web controls

Another way to build web controls is simply by representing them as WebSharper.UI `Doc` values. `Doc` and `Elt` are the main representations of DOM nodes and any dynamic behavior associated with them. The added benefit of building from `Doc` is that the resulting controls can be rendered on the server as well, avoiding the visual flicker we saw earlier with `Web.Control` instances, although more advanced reactive functionality can only be applied from client-side code. More on that on the [UI documentation page](https://developers.websharper.com/docs/v4.x/fs/ui).

Given this, the previous example becomes:

```fsharp
open WebSharper.UI
open WebSharper.UI.Html

type Doc with
    static member MyButton() =
        button [
            on.click (fun e arg -> JS.Alert "clicked!")
        ] [text "Hello world!"]
```

Embedding this control is more straightforward as well, and now the entire control is rendered on the server:

```fsharp
div [] [Doc.MyButton()]
```

## Templating

Writing HTML as inline F# code as shown above is rarely the best approach, because changing the HTML (which you will need more often than you think) requires recompilation. While you can apply seemingly clever tactics to get around having to wait for compilation, such as watching source files and recompiling in the background, or even keeping the compiler in memory to speed up compilation, you are still paying the cost somewhere and your machine is doing mostly unnecessary work. Instead, I usually place HTML into template files and use the templating TP to work with it:

```fsharp
open WebSharper.UI.Templating

type MainTemplate = Template<"main.html", serverLoad=ServerLoad.WhenChanged>
```

This is the more verbose invocation, you can just name the file and be done with it. In the above code, we also want any server-side code to use the latest update to the template, i.e. we should automatically reload the template when it changes. Using the TP relieves you from having to recompile on HTML changes, and instead it will manage refreshing when needed.

The templating TP looks for special WebSharper markers (`ws-..` attributes for content placeholders, event handlers, and model bindings) in the specified source file(s), that you can read about in the "HTML Templates" section of the [UI documentation page](https://developers.websharper.com/docs/v4.x/fs/ui), and exposes various types and members in the resulting type space.

For the CMS functionality in my F# Exchange talk, I built two separate controls - you can see both in the animation above, here are their templates:

1. The "in-place" editor is a simple template with a placeholder for the inner content (`TextContent`) and an `OnAfterRender` (OAR) handler to initialize it according to our needs (planting the `contenteditable` attribute). Saving takes place upon losing focus to keep things simple.

    ```html
    <div class="editor-container" ws-template="InPlaceEditor" ws-onafterrender="OnAfterRender" ws-hole="TextContent"></div>
    ```

2. The plain editor version is slightly more elaborate: it provides a loading indicator so we can tell when it's working in the background, and an actual textarea control to do the editing along with a Save button to send it back to the data store on the server.

    ```html
    <div class="editor-container" ws-template="Editor" ws-onafterrender="OnAfterRender">
      <div class="loader-container ${LoaderCssClass}">
          <div class="loader"></div>
      </div>
      <div class="editor">
        <textarea type="text" ws-var="TextContent"></textarea>
        <div class="overlay">
          <button ws-onclick="Save">Save</button>
        </div>
      </div>
    </div>
    ```

For instance, you could go about instantiating the `InPlaceEditor` template with the folowing code:

```fsharp
MainTemplate.InPlaceEditor()               // Inner template
    .TextContent("Hello")                  // Placeholder
    .OnAfterRender(fun e -> DoSomething()) // Event handler
    ...
    .Doc()                                 // Seal and return as `Doc`
```

Note the type of `e` in the event handler: it's `Runtime.Server.TemplateEvent` instantiated over two types: `MainTemplate.InPlaceEditor.Vars` and `Dom.Event`. You can use the former to access the template model (`.Vars`)- i.e. any model-bound input controls via `ws-var`, the event info (`.Event`), and the target DOM node of the event (`.Target`). These members provide type-safe access to the underlying entities, and they are especially useful to read/populate templated form data.

## The CMS function

It turns out, it's not that easy to just provide a "CMS combinator" that takes any sitelet, enhances its pages with CMS capabilities and returns the result as a new sitelet. Just think of sitelets representing REST endpoints, serving non-HTML content, etc. Instead, to avoid the complexities involved in recognizing sitelet responses, we build on a simpler function to enhance from, one whose signature is:

```fsharp
Context<'T> -> inEdit:bool -> 'T -> Doc
```

Armed with a server-side `render` function of the above signature, we can write our CMS function returning a `Sitelet<CMS<'T>>`:

```fsharp
module CMS =
    type CMS<'T> =
        | [<EndPoint "/edit">] Edit of 'T
        | [<EndPoint "/">] View of 'T

    let AddCMS (render: Context<'T> -> bool -> 'T -> Doc) : Sitelet<CMS<'T>> =
        Sitelet.Infer (fun (ctx: Context<CMS<'T>>) -> function
            | Edit (page: 'T) ->
                Content.Page (render (Context.Map Edit ctx) true page)
            | View (page: 'T) ->
                Content.Page (render (Context.Map View ctx) false page)
        )
```

Here, `'T` is our regular sitelet endpoint type, so we are essentially changing the URL schema of our application from `'T` to `CMS<'T>`, as you'd expect. Next to the original URLs (now mapped over themselves under `View of 'T`), there will now be an `/edit/...` URL for every URL in our inner endpoint type, showing content in "edit" mode. Now, we just need to create such smart content - and that's where our CMS controls come into play.

`AddCMS` simply calls the provided `render` function to render each sitelet page in both edit and view modes, passing down the `inEdit` boolean value, accordingly. This can then be used in the actual render function, and we'll see that shortly.

## Data store

Our data store for CMS content is file-based, we simply stick each key-value pair into a `store` folder (key becomes the file name, value becomes its content.) The names of the keys are important, and they will be determined by the actual CMS control instances.

```fsharp
module MyFileCMS =
    open System.IO

    let Write (sourceFolder, key, newValue) =
        let fname = sprintf "%s/store/%s.html" sourceFolder key
        // Check whether target directory exists, if not, create it
        let dir = Path.GetDirectoryName fname
        if not <| Directory.Exists dir then
            Directory.CreateDirectory dir |> ignore
        File.WriteAllText(fname, newValue)

    let Read (sourceFolder, key) =
        let fname = sprintf "%s/store/%s.html" sourceFolder key
        if File.Exists fname then
            Some <| File.ReadAllText fname
        else
            None
```

## Backend

The server consists of two RPC functions that wrap reading/writing our data store, passing down the root folder of the main web app so the store can read the correct `store` folder:

```fsharp
module Server =
    [<Rpc>]
    let WriteContent (key: string, newValue) =
        let ctx = Web.Remoting.GetContext()
        async {
            return MyFileCMS.Write(ctx.RootFolder, key, newValue)
        }

    [<Rpc>]
    let ReadContent key = 
        let ctx = Web.Remoting.GetContext()
        async {
            return MyFileCMS.Read(ctx.RootFolder, key)
        }
```

## Control #1 - The in-place editor: `Doc.ManagedContentWithInPlaceEditor`

Our in-place editor control has two main chores after rendering itself: registering an `OnBlur` event handler to save the current contents back into our data store (note that we are not doing anything about detecting actual changes, although this would be easy to add) and setting up the `contenteditable` attribute on the parent element.

```fsharp
type Doc with
    /// A UI control that can edit its content on demand.
    /// It must be attached to a container node (p, div, etc.)
    static member ManagedContentWithInPlaceEditor (ctx: Context<_>, inEdit) (key, defValue) =
        let v = MyFileCMS.Read(ctx.RootFolder, key)
        let content = ...
        if inEdit then
            MainTemplate.InPlaceEditor()
                .OnAfterRender(fun (e: Runtime.Server.TemplateEvent<MainTemplate.InPlaceEditor.Vars, _>) ->
                    ...
                    // Auto-save when container/editor node loses focus
                    e.Target.ParentElement.AddEventListener("blur",
                        new System.Action<Dom.Event>(fun e ->
                            let content = ..
                            async {
                                do! Server.WriteContent(key, content)
                            } |> Async.Start
                        ))
                    // Make the parent control editable
                    e.Target.ParentElement.SetAttribute("contenteditable", "true")
                )
                .TextContent(Doc.Verbatim content)
                .Doc()
        else
            if v.IsNone && defValue.IsNone then
                Doc.Empty
            else
                Doc.Verbatim content
```

## Control #2 - The text editor: `Doc.ManagedContent`

Our text editor control has its own CSS, so we add that as a resource to be tracked (you can read more about handling resources on the [Resources page](https://developers.websharper.com/docs/v4.x/fs/resources) in the documentation):

```fsharp
type ManagedContentResource() =
    inherit Resources.BaseResource("WebSharper.CMS.css.ManagedContent.css")
```

The string passed to `BaseResource` above reflects the fact that I have the CMS control in a separate project (`WebSharper.CMS`) with a `css` folder, and a file inside called `ManagedContent.css`. Note the use of `Web.Require` in the control's definition below, taking care of auto-including it whereever we use this control.

```fsharp
type Doc with
    [<Require(typeof<ManagedContentResource>)>]
    static member ManagedContent (ctx: Context<_>, inEdit) (key, defValue) =
        let v = MyFileCMS.Read(ctx.RootFolder, key)
        let content = ...
        if inEdit then
            Doc.Concat [
                MainTemplate.Editor()
                    .OnAfterRender(fun (e: Runtime.Server.TemplateEvent<MainTemplate.Editor.Vars, _>) ->
                        async {
                            ...
                            // Retrieve saved value for our given key
                            let! res = Server.ReadContent key
                            do if res.IsSome then e.Vars.TextContent := res.Value
                            ...
                        }
                        |> Async.Start
                    )
                    .Save(fun (e: Runtime.Server.TemplateEvent<MainTemplate.Editor.Vars, _>) ->
                        ...
                        // Save what's in the editor
                        async {
                            do! Server.WriteContent(key, e.Vars.TextContent.Value)
                            ...
                            ) |> ignore
                        } |> Async.Start
                    )
                    .TextContent(content)
                    .Doc()
                // Add the control's CSS file as a resource.
                Doc.WebControl <| Web.Require(typeof<ManagedContentResource>)
            ]
        else
            if v.IsNone && defValue.IsNone then
                Doc.Empty
            else
                text content
```

## Putting things together

At this point, things go really fast and all we need is to "put everything together." But don't worry, it's quite easy with all the pieces prepared already. First, we need a designer template so our CMS-driven blog looks reasonable.

### Preparing your designer template

I grabbed the excellent and free HTML+CSS template "Mediumish" from free-css.com:

```text
https://www.free-css.com/free-css-templates/page271/mediumish
```

Among others, this comes with a `post.html` and an `assets` folder with JS/CSS/image assets, making it super-easy to reuse. I then planted two placeholders, one for a post's title and one for its content, replacing the text in both with a `ws-hole` attribute on the parent node:

```html
<!-- post.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  ...
  <!-- Make sure asset references start with root -->
  <link href="/assets/css/bootstrap.min.css" rel="stylesheet">
  ...
  <link href="/assets/css/mediumish.css" rel="stylesheet">
</head>
<body>

  <div class="container">
    <div class="row">
      ...
      <!-- Begin Post -->
      <div class="...">
        <div class="...">
          ...
          <!-- Title -->
          <h1 class="posttitle" ws-hole="Title"></h1>
        </div>
        ...
        <!-- Content -->
        <div class="article-post" ws-hole="Content"></div>
        ...
  ...
</body>
</html>
```

We can now bring this template in for our F# needs:

```fsharp
type PostTemplate = Template<"post.html", serverLoad=ServerLoad.WhenChanged>
```

### Building a sitelet for our app

Amazingly, we are just now getting to actually creating the main app. We can model our mini blog with two kinds of pages: a home page and the article/post pages:

```fsharp
type EndPoint =
    | [<EndPoint "/">] Home
    | [<EndPoint "/post">] Post of slug:string
```

Keep in mind, that `AddCMS` expects a `render` function of a particular shape, we can write that function first and simply call `AddCMS` on it:

```fsharp
[<Website>]
let Main =  // Sitelet<CMS<EndPoint>>
    fun ctx (inEdit: bool) -> function
        | Home ->
            ...
        | Post slug ->
            PostTemplate()
                .Title(
                    Doc.ManagedContent (ctx, inEdit) ("title_" + slug, Some <| sprintf "This is blog post %s" slug)
                )
                .Content(
                    Doc.ManagedContentWithInPlaceEditor (ctx, inEdit) ("post_" + slug, None)
                )
                .Doc()
    |> CMS.AddCMS
```

Here we applied our textarea edit control for the title and the in-place editor for the content, and our app now looks like the screenshot in the beginning of this article. (To make this sitelet actually work, `Startup.fs` takes care of embedding it into an ASP.NET Core app.)

Note how the two CMS controls set the names of the keys for title/content for each blog post. For instance, the key for the title content of a blog page whose slug is `3` will be `title_3`, etc. The second argument in that tuple gives a default value, if the key is not yet in the content store.

## Things to explore further

Obviously, we only scratched the surface of what could be accomplished, and here are a few more ideas to try:

1) Markdown content blocks: in addition to storing page content as HTML, you could also provide saving and editing it as Markdown.

2) Adding an offline sitelet for generating a static version of the blog, which could then be uploaded to GitHub Pages, Azure Static Web Apps or other web hosts. You should also check out [SiteFi](https://github.com/granicz/SiteFi) for that.

3) Hosting the main sitelet outside WebSharper, say in Giraffe or Saturn. This is best done with the experimental "general" [sitelets library](https://github.com/IntelliFactory/sitelets).

4) Creating more elaborate editors.

5) Moving the backend data store to a database.

Happy coding and Happy Holidays!
