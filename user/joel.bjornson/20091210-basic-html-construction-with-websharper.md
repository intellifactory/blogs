---
title: "Basic HTML construction with WebSharper"
categories: "f#,websharper"
abstract: "The fundamentals of building HTML (DOM nodes) with WebSharper."
identity: "1914,74746"
---
Regardless the level of sophistication of a web framework, it boils down to the task of generating HTML to be rendered on the client. Therefore I think basic HTML construction is one of the most important things to get right. Plenty of approaches exist, among others:

 * Writing plain HTML
 * Using server side components to be translated into HTML
 * Using XML based markup languages
* Using JavaScript to generate HTML dynamically

In my opinion, neither of these alternatives are completely satisfactory, either because they are not powerful enough, don't compose well, requires an overhead of syntax or lack type safety. Let's have a quick look at the [WebSharper](https://websharper.com) approach. Below is an example of how to construct an HTML node using combinators from the HTML module.

```fsharp
[<JavaScript>]
let MyBook : Element =
    Div [Class "Book"] -< [
        H1 ["The Picture of Dorian Gray"]
        P ["In this book..."]
    ]
```

When compiled with WebSharper, this translates to a JavaScript function for building up the corresponding DOM nodes. Creating reusable templates comes at a very low cost. They are simply defined programatically as functions from a set of parameters to a value of type `Element`:

```fsharp
[<JavaScript>]
let ShowBook (header: string) (body: Element) : Element =        
    Div [Class "Book"] -< [
        H1 [header]
        P [body]
    ]
```

and naturally may be composed with other HTML generating functions:

```fsharp
[<JavaScript>]
let Content =
    Body [
        H1 ["My Books"]
        P [
            ShowBook "The Picture of Dorian Gray" (Span ["..."])
            ShowBook "Bleak House " (Span ["..."])
        ]
    ]
```

Since everything is pure F#, they may also be intermixed with standard F# functions. Here is an example of using `Seq.map` for applying `ShowBook` to a sequence of `Book` elements.

```fsharp
[<JavaScript>]
let Content2 : Element =
    Body [
        H1 ["My Books"]
        P (
            AllBooks()
            |> Seq.map (fun b ->
                ShowBook b.Title b.Description
            )
        )
    ]
```
