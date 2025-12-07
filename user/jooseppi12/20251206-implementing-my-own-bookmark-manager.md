---
title: "Implementing my own bookmark manager"
categories: "f#,websharper,ui,fsadvent"
abstract: ""
identity: "-1,-1"
---

I want to write my own bookmark "manager". Really the most important part is, I want to be able to access sites quickly while I'm on my phone, as the browser's bookmark manager doesn't really cut it for me. On my computer, I do use [buku](https://github.com/jarun/buku). For someone who spends most of their time in the terminal, it's amazing. But that does not really translate to a good experience on mobile. So I want to build a quick prototype, where I can see what I really need on a mobile to conveniently access my bookmarks. To achieve this I'm going to use WebSharper's Offline Sitelets to generate static pages. You can check out the source code [here](https://github.com/Jooseppi12/fsadvent2025) and [here](https://jooseppi12.github.io/fsadvent2025) is a deployed version of the project.

## The storage format

Eventually this will probably be generated from buku's internal database, but for the quick prototype, I'm just going to use a really simple structure to hold the minimal information I need.

```fsharp
type Bookmark =
    {
        url: string
        homepage: bool
        tags: string []
    }
```

I'm adding tags as well, as later on I want to implement some special logic with filtering and also for some default styling for certain categories.

## Home page view

The inspiration for this project came from [this](https://css-tricks.com/hexagons-and-beyond-flexible-responsive-grid-patterns-sans-media-queries/) article on CSS-Tricks. When I saw this, it immediately made me think, that such a layout could be quite interesting to use on phone. CSS has come a long way over the years, so the above layout is entirely possible to achieve without any libraries. So I'm going to generate my layout based on this.

This page is going to be fully rendered on the "server" side, I don't need JS functionality on this page. Let's look at the below html:

```html
<div ws-template="GridItem">
    <a href="${Url}" data-tags="${Tags}">
        <span>${InitialCharacter}</span>
    </a>
</div>
```

The above is going to represent one item on my grid. It's nothing complicated, we just have an `a` tag with a `span` that will show the initial character of the website. Additionally, we have a `data-tags` attribute, that will hold all tags associated with a given bookmark.

So, on the generating process the following function will generate my the whole page for me:

```fsharp
let HomePage ctx (data: Types.Bookmark []) =
    let dataForHomepage = 
        data
        |> Array.filter (fun x -> x.homepage)
    Templating.Main "Homepage" [
        div [] [
            h1 [] [
                a [attr.href "/bookmarks.html"] [text "List"]
            ]
        ]
        div [attr.``class`` "grid"] [
            div [attr.``class`` "container"] [
                dataForHomepage
                |> Array.map (fun x ->
                    let initial =
                        let uri = System.Uri(x.url)
                        // Extract out the initial character
                        uri.Host.Replace("www.", "")[0..0]
                    let tags =
                        x.tags |> String.concat ","
                    Templates.MainTemplate.GridItem()
                        .InitialCharacter(initial)
                        .Tags(tags)
                        .Url(x.url)
                        .Doc()   
                )
                |> Doc.Concat
            ]
        ]
    ]
```

In the above function `data` is just the collection of all bookmarks, that is defined in a json file, that is read on on the start of the static generation process, which is filtered down to the ones, that I want to feature on a quick access view

To give some categorization for the grid layout, we can apply some css rules based on the above defined `data-tags` attribute.

```css
.container div:has(> [data-tags*="programming"]) {
    background: green;
    color: white;
}
```

Using the `*=` check in css, I can define these color rules for some high level tags, creating visual separation for my different sites. So putting all the above together with 

![Home page](https://i.imgur.com/1wlNfQW.png)

## List view

Now let's look at some client functionality. The list of bookmarks is still going to be generated pretty much the same way as for the home page, but instead of showing them in a grid, it will be just a list of urls.

Now to the functionality that requires some JS, the search functionality. This is where tags are going to be useful again for me, as I want to implement two modes: Tag based and text based searching. By default, the search string would be applied on the urls, but if my search string starts with `tag:`, I want to search in the tags. Which should be pretty simple to do:

```fsharp
let searchFunction (s: string) =
    // if text starts with tag:, search within the tags otherwise search in the text url
    let elementsToShow =
        if s.StartsWith "tag:" then
            let tag = s.Replace("tag:", "")
            JS.Document.QuerySelectorAll(sprintf ".bookmark:has(a[data-tags*=\"%s\"])" tag)
        else
            JS.Document.QuerySelectorAll(sprintf ".bookmark:has(a[href*=\"%s\"])" s)
    JS.Document
        .QuerySelectorAll(".bookmark")
        .ForEach((fun (n, _, _, _) -> 
            (As<Dom.Element> n).ClassList.Add "hidden"
        ), null)
    elementsToShow
        .ForEach((fun (n, _, _, _) -> 
            (As<Dom.Element> n).ClassList.Remove "hidden"
        ), null)
```

The above function will either get the elements to show based on the tag search or the normal search mode, hides everything and then only shows what should be based on the search string.

Note, I could have used the search string reactively to dynamically update the list as I type. But I feel like that's not the UX I want from this on the phone, but keeping that option alive as I'm going to test things out in the next couple weeks.

To include this search functionality, I created this template

```html
<div ws-template="Search" class="searchbox">
    <input type="text" ws-var="SearchText"/>
    <button ws-onclick="Search">Search</button>
</div>
```

Which is going to be used like this on the client side code:

```fsharp
let Search () =
    Templates.MainTemplate.Search()
        .SearchText(searchText)
        .Search(fun _ ->
            searchText.View
            |> View.Get searchFunction
        )
        .Doc()
```

Important to note, that this function is in a module that has the `[<JavaScript>]` annotation, so that WebSharper knows to generate JS for the code above!

To include this into the static code generation, we are using the `client` helper function, like this:

```fsharp
client <@ Client.Search () @>
```

With this WebSharper knows the boundary between code that requires functionality from the browser and the rest of the code that is generated on compile time. As the Offline Sitelet during compilation emulates a server behind, it really is a normal WebSharper Sitelet, that on the compilation generates static html from certain, specified endpoints, but more on that below.

![List view](https://i.imgur.com/YkBLSsM.png)

## Static code generation

```fsharp
module Site =
    [<Website>]
    let Main =
        Application.MultiPage (fun ctx action ->
            async {
                let data = System.IO.File.ReadAllText (System.IO.Path.Combine(__SOURCE_DIRECTORY__,"bookmarks.json"))
                let bookmarks = System.Text.Json.JsonSerializer.Deserialize<Types.Bookmark []> data
                return!
                    match action with
                    | Home -> HomePage ctx bookmarks
                    | Bookmarks -> BookmarkPage ctx bookmarks
            }
        )

[<Sealed>]
type Website() =
    interface IWebsite<EndPoint> with
        member this.Sitelet = Site.Main
        member this.Actions = [Home; Bookmarks]
```

WebSharper's Offline Sitelets are based on Standard Sitelets, where we can define what endpoints we want to generate on the Website type's Actions property. This means that in reality, any server side project could be converted a static website easily if they don't have any serverside functionality, that they depend on. But even then, there are tactics you can use to deploy the server separately, but that's something for another time.

## Conclusion

I love using F# as putting together this prototype took less time than creating the static json file, that holds the bookmarks for this demo project. It really shows the power of F# and WebSharper, that a random idea could be turned into a quick prototype in less than 5 minutes.

## What's next?

Well, first I'm going to use this quick demo just to see if I like the idea of a grid like this for the home page. If this style of layout does not work in the end, I might switch to a tile based layout like, similar to what Windows Phone had back in the day. Adding icons instead of using the initial letter of the website (as it could be quite ambiguous). Most importantly though, as mentioned earlier, connecting this with the [buku](https://github.com/jarun/buku) backend would mean that I can easily synchronize this with whatever I'm using from the command line. Additionally as this project grows, probably at some point it will change from being a static website, to have some server side functionality. Like at some point it's probably better to do the searching on the server side, as with a huge amount of bookmarks, dealing with that on the client side could become quite sluggish. But that's for the future, first I need to figure out what UX I really want from this project.

## Final thoughts

I'm looking forward to this year's F# Advent posts to see community solutions and unique problems for inspiration for my own projects.

Thanks Sergey for organizing [F# Advent](https://sergeytihon.com/fsadvent/) year after year!
