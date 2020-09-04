---
title: "CloudSharper 0.9.27 released"
categories: "cloudsharper,fsharp,websharper"
abstract: "CloudSharper 0.9.27 is out, with some UI enhancements and the capability to write WebSharper RPC methods in F# interactive."
identity: "4219,77694"
---
Here comes another release of CloudSharper! Version 0.9.27 comes with a number of enhancements:

* It is now possible to write RPC functions in F# interactive. When you call an `[<Rpc>]`-annotated function from the client side, the arguments are serialized and sent through the WebSocket used by F# Interactive, passed to the function, and the response is sent back the same way.

    Here is an example where two separate widgets share a common server-side state:

    ```fsharp
    let serverSideCounter = ref 0

    [<Rpc>]
    let increment() =
        serverSideCounter := !serverSideCounter + 1

    [<Rpc>]
    let get() =
        async { return !serverSideCounter }

    let x =
        <@ Button [Text "Increment"]
            |>! OnClick (fun _ _ -> increment()) @>

    let y =
        <@ Button [Text "Get the current value"]
            |>! OnClick (fun button _ ->
                async {
                    let! value = get()
                    button.Text <- "The value is " + string value
                }
                |> Async.Start) @>
    ```

* The first time experience is largely improved. When entering CloudSharper without a local component running, instead of a message that could easily be mistaken for an error, CloudSharper now displays a friendly screen with install instructions.

    [![](http://i.imgur.com/qigQ9ug.png)](http://i.imgur.com/ZUkmm2P.png)

* The project templates are updated to the latest WebSharper (3.0.26-alpha).


We are now planning a major rework of the dashboard screen which will contain a lot of cool new features, including GitHub integration and statistics about your workspaces. We can't wait to have them ready for you to enjoy!

Happy coding!
