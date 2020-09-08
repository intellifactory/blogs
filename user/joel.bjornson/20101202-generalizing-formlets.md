---
title: "Generalizing formlets"
categories: "f#,websharper,formlets"
abstract: "This post highlights some of the new features of our redesigned formlet library."
identity: "1908,74740"
---
One of the news of WebSharper 2.0 is a redesigned formlet library. For an introduction to WebSharper formlets, see [Creating web forms using WebSharper formlets](/user/joel.bjornson/20091230-creating-web-forms-using-websharper-formlets.md). And for a running example, have a look at this demo[^1].

The motivation for the new design is twofold:

 * We wanted to address some limitations of our previous implementation (for instance concerning rendering of dynamic forms).
 * We also wished to generalize the formlet concept in order to make it applicable for different domains of GUIs.

However, it is important to stress that the changes to the WebSharper formlet API are minimal. That means that most of the code for old formlets will still work for formlets 2.0.

The most important news is that we introduce a generic formlet interface, parameterized over both the result and body type:

```fsharp
type IFormlet<'B, 'T> =
    abstract member Layout : Layout<'B>
    abstract member Build : unit -> Form<'B, 'T>
    abstract member MapResult : (Result<'T> -> Result<'U>) -> IFormlet<'B,'T>
```

`body` represents the visual part of the form and `result` represents the values being produced. For WebSharper formlets, the body consists of HTML elements. The design enables different implementations of formlets for different contexts (such as GUI environment ) and even removes the dependency on WebSharper for the base library. In fact, the generalized design suits a variety of targets and we have been experimenting with implementations not only for HTML, but also for ExtJs and Windows Presentation Framework (WPF).

Additionally, formlets are now equipped with a layout-manager, responsible for rendering the visual elements of the form.

In consistency with the previous design, formlets contain a function for generating the concrete form parts (making formlets themselves lazy). Another innovation is that form parts now produce streams of edit operations for visually updating forms incrementally. Every time a new edit-instruction is added, it is pushed into the layout-manager, responsible for rendering the formlet. This enables more efficient implementations of layout-managers, by allowing updates of nested forms to only affect the parts that were subject to a change.

In short, some of the merits with the new design include:

## Separation of concerns

Basic formlet combinators are agnostic about the type of the visual component. This makes them reusable for different concrete implementations of formlets. Examples of such functions are `WithLayout`, `Map`, `MapBody`, `Bind`, `SelectMany` and `Sequence`. They constitute the basic building blocks for constructing more specialized ones.

## Flexible layout-managers

Custom layout-managers may be constructed for different rendering purposes. For instance, formlets may be rendered as:

 * Horizontally layed-out components.
 * Vertically layed-out components.
 * Wizard-like step by step interfaces (flowlets)

Here is an example of constructing a custom layout-manager providing a step-by-step interface with a fading effect. The behavior is similar to the rendering mechanism provided by flowlets in the previous formlet implementation:

```fsharp
let FadingFlowletLayout : Layout.Layout<Body> =
    let lm () =
        let panel = Div []            
        // Function for inserting a new element
        let insert (index: int) body =                                
            panel.JQuery.FadeOut("slow", fun _ -> 
                panel.Clear()
                panel.Append(body.Element)
                panel.JQuery.FadeIn("slow", ignore).Ignore
            ).Ignore
            [body.Element]
        { Insert = insert; Panel = panel}
    MakeLayout lm
```

The layout-manager can now be attached to any formlet:

```fsharp
let myFormlet = ...
let myElem =
    myFormlet
    |> Formlet.WithLayout FadingFlowletLayout
```

## Pluggable interfaces

The design of the base library provides a pluggable interface, customizable for different implementations of default layout-managers and reactive-programming operations.

Summarizing our experiences, we are happy to see that formlets as outlined above, forms a very general pattern for describing reactive GUIs!

[^1]: This link is dead and has been removed. You can find formlet examples on [Try WebSharper](https://try.websharper.com), filtering for snippets that use `UI.Formlets`. These formlets are an enhanced, reactive version of the original format library. You can find more information the [WebSharper.Forms README](https://github.com/dotnet-websharper/forms).
