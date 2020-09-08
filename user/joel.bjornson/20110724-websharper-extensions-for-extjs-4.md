---
title: "WebSharper Extensions for ExtJS 4"
categories: "f#,websharper,extjs"
abstract: "In this post I introduce our latest WebSharper Extensions for ExtJS 4."
identity: "1904,74736"
---
The latest [WebSharper Extensions for ExtJS](https://developers.websharper.com/docs/v4.x/fs/extjs)[^1] (version 2.3.17) contain a native binding to the JavaScript library ExtJS 4. [Ext JS](https://www.sencha.com/products/extjs/) is a popular JavaScript framework for creating rich client-side applications supporting a large set of desktop-like widgets.

The WebSharper Extensions allow you to develop ExtJS 4 applications with WebSharper using a typed F# API.

In order to try it out yourself you need to:

 * Download and install WebSharper 2.3 from [websharper.com](https://websharper.com).
 * Download and install the WebSharper Extensions for ExtJS.
 * Start a new WebSharper project (for example a WebSharper 2.3 HTML Application) and include a reference to the ExtJS dll.

Following is an example of a simple ExtJS widget defined with WebSharper. The sample consists of a panel with a button. When the button is clicked, an alert message is displayed. The panel is rendered inside a regular `DIV` element by hooking into the `OnAfterRender` event:

```fsharp
[<JavaScript>]
let Sample () =
    Ext.panel.Panel (
        Ext.panel.PanelConfiguration (
            Border = false,
            Padding = 20.,
            Items = [|
                Ext.button.ButtonConfiguration (
                    Xtype = "button",
                    Text = "Click",
                    Handler = (fun _ ->
                        Ext.Msg.Alert("Alert","Message alerted!")
                    )
                )
            |]
        )
    )

[<JavaScript>]
let Main () =
    Div []
    |>! OnAfterRender (fun el ->
        Ext.OnReady(fun _ ->
            let sample = Sample ()
            sample.Render(el.Body)
        )
    )
```

The Main `DIV` is rendered in the browser as:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=204)

ExtJS applications written in WebSharper can naturally benefit from common WebSharper functionality such as client side HTML construction. The following helper function may be used to embed HTML elements inside an ExtJS container:

```fsharp
[<JavaScript>]
let ElementContainer (element: Html.Element) =
    Ext.container.Container (
        Ext.container.ContainerConfiguration(
            ContentDom = element.Body
        )
    )
```

Here is an example of rendering HTML elements inside a tabs widget:

```fsharp
[<JavaScript>]
let TabsSample () =
    Ext.tab.Panel (
        new Ext.tab.PanelConfiguration(
            Padding = 20.,
            Width = 300. , 
            Height = 200. ,
            Items = 
                [|
                    Ext.panel.PanelConfiguration (
                        Title = "Tab 1",
                        BodyPadding = 10.,
                        Items = [|ElementContainer(H1 [Text "Tab 1 Content"] )|]
                    )
                    Ext.panel.PanelConfiguration (
                        Title = "Tab 2",
                        BodyPadding = 10.,
                        Items = [|
                            ElementContainer(
                                Div [
                                    H1 [Text "Tab 2 Content"] 
                                    P [Text "Content goes here"]
                                ]
                            )
                        |]
                    )
                |] 
        )
    )
```

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=205)

The F# API binding is based on annotations from the ExtJS source code, in a simliar way that the official [API Documentation](https://docs.sencha.com/)[^2] is generated.

In cases where annotations are missing or spelled out incorrectly, the documentation (and the WebSharper binding) fails to accurately map the intended API. This may be appearant when trying to convert some JavaScript samples to WebSharper.

To give a fictional example, assume you want to pass a configuration option foo to an ExtJS class lacking that property. For example, setting `foo` for `Ext.tree.Panel` fails to compile:

```fsharp
let tree =
    Ext.tree.Panel (
        Ext.tree.PanelConfiguration(
            Title = "Tree Panel",
            Height = 200.,
            Store = store,
            foo = "foo" // Invalid property
        )
    )
```

If you are still convinced that the property indeed is feasible, there are a few workarounds available. For example, you can create a new extended `Ext.tree.PanelConfiguration` type defining the extra property:

```fsharp
type TreePanelConfiguration() =
    inherit Ext.tree.PanelConfiguration()

    [<DefaultValue>]
    val mutable Foo : string

let tree =
    Ext.tree.Panel (
        TreePanelConfiguration (
            Title = "Tree Panel",
            Height = 200.,
            Store = store,
            foo = "foo"
        )
    )
```

You may also resort to a dynamic style of programming; Either using the new support for JavaScript object literal expressions as in:

```fsharp
let tree =
    Ext.tree.Panel (
        New<Ext.tree.PanelConfiguration> [
            "title" => "Tree Panel"
            "height" => 200
            "store" => store
            "foo" => "foo"
        ]
    )
```

or by enhancing an existing configuration object:

```fsharp
let tree =
    let conf =
        Ext.tree.PanelConfiguration(
            Title = "Tree Panel",
            Height = 200.,
            Store = store
        )
    conf?foo <- "foo"
    Ext.tree.Panel(conf)
```

This style of programming is not desirable (given the lack of static type checking) but at least it gives you the same capabilities as writing pure JavaScript when needed.

To see some live examples of ExtJS programming with WebSharper, check out the demo section[^3].


[^1]: The original link is dead. It is now updated to point to the WebSharper.ExtJS 4.x documentation page instead.

[^2]: Link has been updated to point to the latest API documentation. Original link referenced the 4.0 docs.

[^3]: This link is dead and has been removed. You can find ExtJS examples on [Try WebSharper](https://try.websharper.com), filtering for snippets that use `ExtJS`.
