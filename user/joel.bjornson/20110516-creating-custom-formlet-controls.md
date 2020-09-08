---
title: "Creating Custom Formlet Controls"
categories: "f#,websharper,formlets"
abstract: "This post shows how to create custom formlet controls."
identity: "834,74566"
---
Sometimes the standard formlet controls are not sufficient for constructing particular input UI components. For example you may want to include some third party UI widget in your forms or need exact control over the internal HTML components.

The function `BuildFormlet` enables you to create arbitrary formlet controls by supplying a function for generating:

 * An element representing the body of the form.
 * A function for resetting the form.
 * An `IObservable` representing the logical state of the form.

Here is an example of constructing a text box formlet (similar to `Controls.Input`):

```fsharp
[<JavaScript>]
let Text (initValue: string) =
    Formlet.BuildFormlet <| fun () ->
        let state = new Event<_>()
        let trigger value =
            state.Trigger (Result.Success value)
        let body = 
            Input [Attr.Value initValue]
            |>! Events.OnKeyUp (fun input _ ->
                trigger input.Value
            )
            |>! OnAfterRender (fun _ ->
                trigger initValue
            )
        let reset () =
            body.Value <- initValue  
            trigger initValue
        body, reset, state.Publish
```

The body consists of an input element initialized with a default value. The state is constructed by an event that is triggered with a the current value of the text box everyime a key is released. The reset function reinitializes the text box to it's default value and triggers a corresponding state change.

When defining custom formlets there are a few questions you need to consider:

 * Is the formlet initialized correctly (both with respect to its logical and visual state)?
 * Does an invocation of reset trigger the initial value?
 * Does the logical state always reflect the visual state?

To simplify testing of your custom controls it is useful to define a combinator for visulazing the state of a formlet. The following function enhances a formlet with a panel for displaying the result and attaching a reset button:

```fsharp
[<JavaScript>]
let TestFormlet (formlet: Formlet<string>) =
    Formlet.Do {
        let! res = 
            formlet
            |> Formlet.LiftResult
        return!
            Formlet.OfElement <| fun _ ->
                let status =
                    match res with
                    | Result.Success x ->
                        "Success : " + x
                    | Result.Failure fs ->
                        "Failure: " + (Seq.fold (+) "" fs)  
                Div 
                    [Attr.Style "border:2px solid #CCC; padding:10px; margin-top:10px"] 
                    -< [Text status]
    }    
    |> Enhance.WithResetButton   
    |> Enhance.WithFormContainer  
```

For example you can enhance the `TextBox` from above using the `TestFormlet` combinator:

```fsharp
TestFormlet (TextBox "")
```

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=189)

Following is a slightly more advance example of menu control implemented as a custom formlet. Functionality wise the menu is similar to `Controls.Select`, but with a different user interface. Given a default index and a list of label and value pairs, the a menu formlet is constructed, displaying a list of clickable items. The logical state of the formlet should indicte the corresponding value of the last clicked item:

```fsharp
[<JavaScript>]
let Menu (defIndex: int) (items: list<string * 'T>) =
    Formlet.BuildFormlet <| fun () ->
        let update = new Event<int>()
        // Body is the list.
        let body =
            UL [Attr.Class "menu"] -< (
                items
                |> List.mapi (fun index (label, value) ->
                    LI [Text label]
                    |>! OnClick (fun _ _ -> update.Trigger(index))
                    |>! OnAfterRender (fun item ->
                        // Subscribte to the update event.
                        update.Publish.Subscribe(fun updateIndex ->
                            if index = updateIndex then
                                item.AddClass "active"
                            else
                                item.RemoveClass "active" 
                        )
                        |> ignore
                    )
                )
            )
            |>! OnAfterRender (fun _ -> update.Trigger defIndex)
        // State constructed by mapping the update event.
        let state =
            let values =
                items
                |> List.map (fun (_,value) -> value)
                |> List.toArray
            update.Publish
            |> Event.map (fun index ->
                Result.Success values.[index]
            )                    
        // Reset triggers an update for the initial index.
        let reset () = 
            update.Trigger defIndex
        body, reset, state
```

As with any other formlet the Menu formlet can now be composed with other formlets or enhanced with validation etc.

Here is an example of embedding a menu component in a composed formlet:

```fsharp
[<JavaScript>]
let MyForm =
    let name =
        Controls.Input ""
        |> Enhance.WithTextLabel "Name"
                    
    let menu =
        [
            "Alpha", "A"
            "Beta", "B"
            "Gamma" , "G"
            "Delta", "D"
            "Epsilon" , "E"
        ]
        |> MenuFormlet 0 
        |> Enhance.WithTextLabel "Select"

    Formlet.Yield (fun _ _ -> ())
    <*> name <*> menu                                
    |> Enhance.WithSubmitAndResetButtons
    |> Enhance.WithFormContainer
```

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=190)

Even rather complex UI controls can typically be lifted into formlets. There are several advantages of doing that. First of all it enforces you to have a clear understanding of what logical entities your UI widgets represent. Further it enforces a seperation of the logical and visual state of a control and more importantly it isolates the logic of dealing with internal state changes and hides this implementation for the consumer of your formlet widget. By exposing a widget as a formlet you also get all the functionality for composition, validation and enhancements for free.
