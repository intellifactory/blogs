---
title: "An asynchronous Slot machine"
categories: "f#,websharper"
abstract: "In this post I highlight the enhanced support for asynchronous programming in WebSharper. I show an example, modelling a slot machine as asynchronous computations run in parallel."
identity: "1905,74737"
---
One of the new features in WebSharper 2.3 is extended support for asynchronous programming on the client. The new implementation of `Async` uses a round-robin scheduler to alternate between computations and mimic parallel execution.

Async is basically implemented as a continuation monad where the definition of bind delays the invocation of the continuation by scheduling it for execution.

This way we can support the functions `Async.Parallel` and `Async.StartChild` with similar semantics to their F# counterparts.

To give an example of a use case for `Async.Parallel`, below is an application modeling a slot machine as computations run in parallel. The idea is to have three different widgets, each alternating between a series of images with a small time delay. When an image is clicked, the alternation stops and the current image is displayed.

The objective is to select the same image icon for all slots. When done, a message is displayed showing whether the result was successful or not:

> **Review note** <br />
> This resource needs to be re-added.
> ![](//www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=200)

> **Review note** <br />
> This resource needs to be re-added.
> ![](//www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=199)

The icons are images stored in an image folder:

```fsharp
[<JavaScript>]
let Icons = 
    [|
        "images/cherry.png"
        "images/bell.png"
        "images/bar.png"
        "images/orange.png"
        "images/seven.png"
        "images/bomb.png"
    |]
```

A function `Slot` that returns an image element and a computation that can be invoked to start the image alternation is defined as:

```fsharp
[<JavaScript>]
let Slot () =
    let clicked = ref false
    // Returns the url to the next icon image.
    let nextIcon = 
        let n = 
            Math.Random () * (float Icons.Length)
            |> Math.Floor
            |> ref
        fun () -> 
            incr n
            Icons.[!n % Icons.Length]

    // The image displaying an icon.
    let img = 
        Img [Attr.Class "slot"; Src Icons.[0]] 
        |>! OnMouseDown(fun img _ -> 
            img.AddClass "done"
            clicked := true
        )
    // The computation for alternating the image src.
    // The computation returns the current image src
    // once the the value of clicked is set to true.
    let run =
        let rec loop () =
            if clicked.Value then
                async.Return <| img.GetAttribute("src")
            else
                async {
                    do img.["src"] <-  nextIcon ()
                    do! Async.Sleep 200
                    return! loop ()
                }
        async {
            // Small initial delay.
            do! Async.Sleep ( Math.Round <| Math.Random () * 1000.)
            return! loop ()
        }
    img, run
```

Once the image is clicked, the computation returns the `src` attribute of the current image.

Now, a function `SlotMachine` can be implemented using `Async.Parallel` to run three instances of the `Slot` widget in parallel:

```fsharp
let SlotMachine () =
    let slots = List.init 3 Slot
    let res = Div [Attr.Class "result"]
    Div (List.map fst slots) -< [res]
    |>! OnAfterRender (fun _ ->
        async {                
            // Run the three slots in parallel
            let! xs = Async.Parallel (List.map snd slots)
            
            // Check if all are similar
            if (Set.ofArray xs).Count = 1 then
                res.AddClass "success"
                res.Append "Congratulation!"
            else
                res.AddClass "fail"
                res.Append "Not quite right."
            return ()
        }
        |> Async.Start
    )
```

Note that the results of the three slot computations are bound to `xs` and that they are run in parallel rather than in a sequence. When all of the widgets returned, either a success or failure message is displayed depending on whether the same icon was selected for all widgets or not.
