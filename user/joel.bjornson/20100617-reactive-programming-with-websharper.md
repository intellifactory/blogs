---
title: "Reactive programming with WebSharper"
categories: "f#,websharper,reactive"
abstract: "A small example highlighting F# (and WebSharper) support for asynchronous and reactive programming."
identity: "1911,74743"
---
In following post I'd like to highlight F# (and WebSharper) support for asynchronous and reactive programming.

Some of the things that make F#/WebSharper particularly suited for this domain includes:

 * Events as first class values
 * Provided event combinators - mapping, merging, filtering etc.
 * Nice syntax for asynchronous programming via work-flow builders.

To give a concrete example let's implement a small application polling the server for price information. First we assume a server side method that given a string value representing a country returns the current price:

```fsharp
[<Rpc>]
let CurrentPrice (country: string) : Async<int>=        
    async {
        let price = ...
        return price
    }
```

In order for the function to be non blocking the actual calculation of the price information is encapsulated within an asynchronous computation.

We are interested in keeping track of the highest price and showing the maximum value found so far by any call to the server method.

We also assume that the pricing information fluctuates frequently, why we need to poll the server repeatedly.

A sequence of prices may be modeled as an event stream. Here is a general helper function for creating a stream of values, given a function for fetching a new result and a delay time for specifying the sampling frequency:

```fsharp
[<JavaScript>]
let ValueStream (delay: int) (getValue : unit -> Async<'T>) : IEvent<'T> = 
    let resultEv = Event<'T>() 
    let rec work () = async {
        let! x = getValue ()
        resultEv.Trigger x
        do! Async.Sleep delay    
        do! work ()
    }                
    work ()
    |> Async.Start
    resultEv.Publish
```

First a new event (`resultEv`) is created. Then a function (`work`) producing an asynchronous value is defined. The resulting value is pushed into `resultEv` as a side effect.

Next, the `Main` function defines a list of value streams based on different ways to call the `CurrentPrice` method and creates a dynamically updated element displaying the latest price information:

```fsharp
[<JavaScript>]
let Main () =
    let bestPriceLabel =
        [
            ValueStream 3000 (fun () -> CurrentPrice "Hungary")
            ValueStream 2000 (fun () -> CurrentPrice "France")
            ValueStream 1000 (fun () -> CurrentPrice "USA")
        ]
        |> List.reduce Event.merge
        |> Event.scan max 0
        |> Event.pairwise
        |> Event.choose (fun (x,y) -> 
            if y > x then 
                Label [
                    Text ("Best Price: " + string y)
                ]
                |> Some
            else 
                None
        )
    Div []
    |>! OnAfterRender (fun el ->
        bestPriceLabel.Add (fun label ->
            el.Clear()
            el.Append label
         )       
    )
```

The list of events is merged into a single event stream. `Event.scan` is then used to accumulate the maximum price found so far. Since we are only interested in new values the `pairwise` and `choose` functions are utilized in order to only collect values larger than the previous one. The integer values are transformed into `Html.Element` objects, producing a stream of elements.

Finally to show the labels we subscribe to the `bestPriceLabel` event with a callback function for appending labels to a container element.

Any time a new (higher) price is encountered, the content of the outer container will be replaced with a new label displaying the price.
