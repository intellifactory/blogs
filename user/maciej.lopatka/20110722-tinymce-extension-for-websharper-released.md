---
title: "TinyMCE Extension for WebSharper Released"
categories: "f#,websharper,tinymce"
abstract: "We are happy to announce a new WebSharper extension, this time for TinyMCE. TinyMCE is one of the most popular Javascript WYSIWYG editors."
identity: "1045,74626"
---
We are happy to announce a new WebSharper extension, this time for TinyMCE. TinyMCE is one of the most popular Javascript WYSIWYG editors.

It provides a powerful HTML editor supporting:

 * Basic text editing
 * Embedding images
 * Creating tables
 * Defining custom plugins

The WebSharper extension includes:

 * A direct mapping of the TinyMCE API
 * A formlet extension for embedding TinyMCE widgets as formlet controls


## Using the formlet controls

The simplest way to include a TinyMCE editor in your WebSharper application is to use one of the two formlet controls:

 * `Controls.SimpleHtmlEditor` - A basic control with limited configuration options.
 * `Controls.AdvancedHtmlEditor` - An advanced control with more more configuration options.

The following code snippet shows how to create an `AdvancedHtmlEditor` formlet with a toolbar menu at the top and aligned to the left:

```fsharp
[<JavaScript>]
let Editor () : Formlet<string> =
    let conf =
        { AdvancedHtmlEditorConfiguration.Default with
            Width = Some 600
            Height = Some 400
            ToolbarLocation = Some ToolbarLocation.Top
            ToolbarAlign = Some ToolbarAlign.Left
        }
    Controls.AdvancedHtmlEditor conf "default"
    |> Enhance.WithSubmitAndResetButtons
```

Rendered in a browser as:

> **Review note** <br />
> This resource needs to be re-added.
> ![](//www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=206)

## Using the direct binding

Instead of using the formlet interface you can create the corresponding example using the direct binding for TinyMCE as well. This gives you more fine grained control over the configuration options and behavior.

The following example creates a TinyMCE editor with a custom handler for click events.

```fsharp
let Editor () = 
    TextArea [Text "Default  text"]
    |>! OnAfterRender (fun el ->
        TinyMCE.Init (
            new TinyMCEConfiguration(
                Theme = "simple",
                Mode = Mode.Textareas,
                Oninit = (fun () ->
                    let editor = TinyMCE.Get(el.Id)
                    editor.SetContent("New content") |> ignore
                    editor.OnClick.Add (fun (ed:Editor) ->
                        JavaScript.Alert(ed.GetContent())
                    ) |> ignore
                )
            )
        )
    )
```

Displayed as:

> **Review note** <br />
> This resource needs to be re-added.
> ![](//www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=202)

You can download the WebSharper Extension for TinyMCE at this address[^1].


[^1]: Link is dead and was removed. Check the [NuGet.org/WebSharper](https://www.nuget.org/packages?q=websharper) page for a list of public WebSharper extensions.
