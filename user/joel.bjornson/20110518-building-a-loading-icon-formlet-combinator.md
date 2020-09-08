---
title: "Building a Loading Icon Formlet Combinator"
categories: "f#,websharper,formlets"
abstract: "In this post I show how to implement a formlet combinator encapsulating the common AJAX pattern of displaying a loading icon while submitting values to the server and waiting for a response."
identity: "833,74565"
---
In my [previous post](/user/joel.bjornson/20110516-creating-custom-formlet-controls.md) I show how to create custom formlets using `Formlet.BuildFormlet`. Continuing on the same theme, here is an example of how to create a custom formlet combinator for capturing the common AJAX pattern of submitting data to the server and displaying a loading icon while awaiting the result. The scenario to be addressed is:

 * A client side formlet produces a value.
 * The value is posted to the server and a loading icon is displayed.
 * The server returns and the loading panel is removed.
 * Depending on the result of the returned value, the continuation of the form is displayed.

Prefarably the interface should be non-blocking. This is acchieved by making RPC server side function return `async` values.

Since the server call can be represented as an arbitrary asynchronous computiation, we can simply define a combinator that lifts an asyncrounous value into a formlet, displaying a loading panel until the computation is done:

```fsharp
[<JavaScript>]
let LoadingFormlet (a:Async<'T>) : Formlet<'T> =
    let loadingPane =
        Formlet.BuildFormlet <| fun _ ->
            let elem = Div [Attr.Class "loadingPane"]
            let state = new Event<_>()
            async {
                let! value = a
                do state.Trigger (Result.Success value)
                return ()
            }
            |> Async.Start
            elem, ignore, state.Publish    
    Formlet.Replace loadingPane ( fun value -> 
        Formlet.Empty () 
        |>  Formlet.InitWith value
    )
```

The `loadingPane` formlet is constructed as an custom formlet, using `Formlet.BuildFormlet`. The body part is a `div` element with a special class attribute to allow styling of the panel. The state is an event that is triggered as soon as the `async` computation produces a value.

The function `Formlet.Replace` is used to create a formlet that replaces the loading panel with an empty formlet once the value is triggered.

Using the combinator is straightforward. Here is an example:

```fsharp
[<JavaScript>]
let MyForm =
    Formlet.Do {
        // Get the value of the form
        let! message = 
            Controls.TextArea ""
            |> Enhance.WithTextLabel "Message"
            |> Enhance.WithLabelAbove
            |> Enhance.WithSubmitButton

        // Call the server
        let! res = LoadingFormlet (Post message)
        
        // Display the server result 
        return! 
            Formlet.OfElement (fun _ ->
                Div [Attr.Class "info"] -< [Text res]
            )
    }        
    |> Enhance.WithFormContainer
```

And the resulting form:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=191)

After submitting the form the loading panel is displayed:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=192)

When the server returned:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=193)
