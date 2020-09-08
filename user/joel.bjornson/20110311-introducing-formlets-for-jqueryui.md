---
title: "Introducing formlets for jQueryUI"
categories: "f#,websharper,formlets,jqueryui"
abstract: "In this post I introduce our latest WebSharper extension - Formlets for jQueryUI. "
identity: "843,74567"
---
The WebSharper Formlets library provides powerful abstractions for constructing and composing forms in a declarative style. However, the standard library only exposes the basic form elements (text boxes, select boxes, text areas etc) as primitive formlet controls.

[jQueryUI](https://jqueryui.com/) is a JavaScript library offering more advanced UI widgets and effects such as Tabs, Sliders, Drag & Drop functionality, and Accordions.

In our latest WebSharper extensions, we merge the two by supplying a formlet interface for jQueryUI widgets. Our aim is to make it *as simple as ever possible* to create forms containing useful widgets such as calendars for selecting dates, tabs for choosing between different input forms or sortable lists for enabling ordering of items.

Here is a short example utilizing some of the features of WebSharper formlets and the jQueryUI formlet extension.

The example consists of four formlets with internal dependencies:

 * A formlet for inputting a set of labels (string values).
 * A formlet for sorting the list of labels.
 * A formlet for selecting a subset of the labels using Drag & Drop.
 * A Formlet showing an input text box with autocompletion based on the selected labels.

```fsharp
[<JavaScript>]
let LabelsForm =
    Formlet.Do {
        let! labels = 
            Controls.Input "" 
            |> Validator.IsNotEmpty "Empty value not allowed"
            |> Enhance.WithValidationIcon
            |> Enhance.Many
            |> Enhance.WithLegend "Add Label"
            |> Enhance.WithSubmitAndResetButtons "Test" "Reset"
        let! orderedLabels =
            labels
            |> List.map (fun label ->
                Formlet.OfElement (fun _ -> Label [Text label])
                |> Formlet.Map (fun _ -> label)
            )
            |> Controls.Sortable
            |> Enhance.WithLegend "Order"
        let! selectedLabels =
            orderedLabels
            |> List.map (fun label ->
                label, label, false
            )
            |> Controls.DragAndDrop None
            |> Enhance.WithLegend "Drag & Drop"
        return! 
            Controls.Autocomplete "" selectedLabels
            |> Enhance.WithTextLabel "Label"
            |> Enhance.WithLegend "Autocomplete"
    }
    |> Formlet.Map ignore
```

Here is the above formlet rendered in a browser:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=174)

It first lets the user input a sequence of labels (enabled by the `Enhance.Many` combinator).

Once submitted, the three dependent formlets are rendered:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=175)

The first formlet provides a capability to sort the list of labels by dragging them. Depending on the specified order, the subsequent formlet allows the user to select a subset of the labels by Drag & Drop.

Finally, the last formlet uses the selected labels to create a text box enhanced with autocompletion for showing matching suggestions as the user is typing.

The look and feel of the jQueryUI widgets (and other formlet icons) are configurable by overriding the jQueryUI CSS. Adding the following line to web.config changes the theme to (in this case to `flick`):

```xml
<add key="IntelliFactory.WebSharper.JQueryUI.Dependencies+JQueryUICss" 
     value="http://ajax.googleapis.com/ajax/libs/jqueryui/1.8.1/themes/flick"></add>
```

Here is the same formlet rendered with the new theme:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=176)
