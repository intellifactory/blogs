---
title: "Dart in F#, differently"
categories: "f#,websharper,ui,canvas,fsadvent"
abstract: ""
identity: "-1,-1"
---

It's hard to believe we are getting this close to the end of the year, but it means that one of my favourite sporting event of the year is getting closer: [the PDC World Darts Championship][1] (to be more precise, it's starting today!). Darts is probably one of the most followed sport in recent years for me and it's not a secret that at some point in my life, I would to attend this event live, as the atmosphere that crowd creates gives me goosebumps even through the TV broadcasting.

One of the first projects that I have ever created was a browser based Darts calculator. It was printing out all the possible combinations you can do for a given number with 3 dart throws into a div. Nostalgia hit me and decided to take a look back at that project, just to find how bad of an implementation that was.

You can find the project on [GitHub][6] and a live demo [here][7].

## Why was it a bad implementation?

Well, it was quite an ineffective implementation. It generated all possible combinations a player could achieve with 3 darts at most and then filtered out those combinations that did not reach the sum. Also, other than the number input from the user, there was not much to interact with as the results were displayed as plain text. It was not even able to filter out the [non-checkout][2] combinations. Now with a few years of experience I feel like I can create a better one and why would I not share it with the community?

## Let's start with the basics

The user should be able to provide the number they want to reach with a maximum of 3 throws and if they are going for a checkout. If they are going for a checkout, that means we should not return any combination, that would not end with hit on the double ring of the board.

To represent a single dart throw I used the following type:

```fsharp
type Dart =
    | Triple of int
    | Double of int
    | Single of int
    | Nil
```

The interesting bit here is including Nil as part of my representation, but more on that later when we get to the generator logic.

I also provided some helper functions to pretty print a dart throw (i.e., a triple 20 is being shown as T20, doubles are shown as D20 and so on), to get the scoring value of a dart throw etc.

Now on to generating all the results. To make my life easier, I have created a `Map`, that stores all possible values that you can score with a single dart. So, an example key-value pair of this map would be `(12, [Triple 4; Double 6; Single 12])`, as all values in the list have a scoring value of 12.

Then once we have this Map (named `scoringValues`), calculating out all possible combinations becomes quite a bit easier, as instead of having three different entries for the value of 12, I can just work with a single value here and at the end we can do the 1-X mapping.

Now with that in place, let's look at the function:

```fsharp
let getCombinationsFor (numberToReach: int) (isItCheckout: bool) =
    scoringValues.Keys
    |> List.ofSeq
    |> List.choose (fun key ->
        if key = 0 && numberToReach <> 0 then
            None
        else if numberToReach - key < 0 then
            None
        else
            if isItCheckout && numberToReach - key = 0 && key % 2 = 0 then
                Some (key, numberToReach - key)
            else if isItCheckout && numberToReach - key = 0 then
                None
            else
                Some (key, numberToReach - key)
    )
    |> List.collect (fun (v1, remainder) ->
        scoringValues.Keys
        |> List.ofSeq
        |> List.choose (fun key ->
            if key = 0 && remainder <> 0 then
                None
            else if remainder - key < 0 then
                None
            else
                if remainder - key = 0 && isItCheckout && key % 2 = 0 then
                    Some (v1, key, remainder - key)
                else if remainder - key = 0 && isItCheckout then
                    None
                else
                    if scoringValues.ContainsKey (remainder - key) && (not isItCheckout || (remainder - key) % 2 = 0) then
                        Some (v1, key, remainder - key)
                    else
                        None
        )
    )
    |> List.collect (fun (d1, d2, d3) ->
        let getFilteredByCheckout (items: Dart list) =
            if isItCheckout then
                items |> List.filter Dart.IsCheckout
            else
                items
        match scoringValues.TryGetValue(d1), scoringValues.TryGetValue(d2), scoringValues.TryGetValue(d3) with
        | (true, items1), (true, [Nil]), (true, [Nil]) ->
            getFilteredByCheckout items1 |> List.map (fun x -> Some (x, Nil, Nil))
        | (true, items1), (true, items2), (true, [Nil]) ->
            items1 |> List.collect (fun x -> getFilteredByCheckout items2 |> List.map (fun y -> Some (x, y, Nil)))
        | (true, items1), (true, items2), (true, items3) ->
            items1 |> List.collect (fun x -> items2 |> List.collect (fun y -> getFilteredByCheckout items3 |> List.map (fun z -> Some (x, y, z))))
        | _ -> []
    )
    |> List.sortBy (fun (Some (a, b, _)) -> a, b)
```

To dissect this function let's go step-by-step:

The first `List.choose` gets us the value of the first dart throw. If our desired value is the same as the first dart throw value and we require a checkout, we are also checking if we are even or not. If we are odd, we don't care about this result, as it is not possible to achieve a checkout on an odd number with a single throw.

The first `List.collect` is getting us the second and third throws. This is where having Nil as part of the type representation comes handy, as with this we are not dealing with variability of needing 1, 2 or 3 dart throws to achieve the score. At this level we are handling everything as a combination of three throws and the display layer will handle showing what is an actual throw or not.

The last `List.collect` is doing the logic of the previously mentioned 1-X mapping logic, where if we have the value of 12 and let's say we did not require checkout, we should end up with `T4`, `D6` and `12` as all possible combinations.

## What about displaying the results for the user?

In my original implementation, this was probably the worst part, as the user was only presented with a wall-of-text of all possible combinations. So let's see what we can do to make this a bit more interactive. I'm going to use [WebSharper.UI][3]'s reactive layer in combination with the standard [Canvas][4] from JS

First thing is I would like to introduce a darts board that is rendered via the browser with the help of the canvas element.

![Darts board with a highlighted combination](https://i.imgur.com/n5B2lF1.gif "Darts board with a highlighted combination")

Drawing a darts board with canvas is relatively simple, as we can boil it down to 3 different problems:

- Drawing circles
- Drawing sectors
- Printing the label

### Drawing circles

Drawing circles is straightforward with canvas. You only need to provide the center point, the radius of the circle and range of angles (in radian) you want to render. As we want to draw a full circle here, we would use 0 to 2 * π

```fsharp
let renderCircle (ctx: CanvasRenderingContext2D) radius color = 
    ctx.BeginPath();
    ctx.Arc(WIDTH/2., HEIGHT/2., radius, 0, 2. * Math.PI, false)
    ctx.LineWidth <- 1.
    ctx.StrokeStyle <- color
    ctx.FillStyle <- color
    ctx.Fill()
    ctx.Stroke()
```

### Drawing sectors

Based on the above, drawing a sector is also simple, as we can just provide an angle range that we want to get rendered. But one thing to note is that the 0 angle is on the X axis. We also know that each sector is 18° (20 numbers spread across a full circle).

```fsharp
let renderSectorPart (ctx: CanvasRenderingContext2D) radius color (nth: int) = 
    let beginDeg = (-99 + nth * 18) // We are starting to render from the value of 20 clockwise
    let beginRad = convertToRadian beginDeg
    let endRad = convertToRadian (beginDeg + 18)
    ctx.BeginPath();
    ctx.MoveTo(WIDTH/2., HEIGHT/2.)
    ctx.Arc(WIDTH/2., HEIGHT/2., radius, beginRad, endRad, false)
    ctx.LineWidth <- 1.
    ctx.StrokeStyle <- color
    ctx.LineTo(WIDTH/2., HEIGHT/2.)
    ctx.FillStyle <- color
    ctx.Fill()
    ctx.Stroke()
```

### Printing the label

This is probably the most complicated part of the canvas rendering, as it involves rendering text + rotating the text, but we can achieve a ring of text rendering with some knowledge we have from the circle rendering above. The trickiest part is probably the rotation of the bottom half of the labels to face the correct way, but the function below should do the trick:


```fsharp
let renderLabel (ctx: CanvasRenderingContext2D) (num: int) (nth: int) =
    ctx.Save()
    ctx.StrokeStyle <- "white"
    ctx.FillStyle <- "white"
    ctx.Font <- "30px Arial"
    ctx.TextAlign <- CanvasTextAlign.Center
    ctx.Translate(WIDTH/2., HEIGHT/2.)
    // The second quadrant should match the orientation of the fourth one
    if nth > 5 && nth < 10 then
        ctx.Rotate(convertToRadian ((20 + nth - 10) * 18))
    // 3 should match the orientation of 20
    else if nth = 10 then
        ctx.Rotate(convertToRadian 0)
    // The third quadrant should match the orientation of the first one
    else if nth > 10 && nth < 15 then
        ctx.Rotate(convertToRadian ((nth - 10) * 18))
    else
        ctx.Rotate(convertToRadian (nth * 18))
    //ctx.Rotate(convertToRadian (nth * 18))
    ctx.FillText(string num, 0, if nth > 5 && nth < 15 then HEIGHT/2. * 0.85 + 15. else - HEIGHT/2. * 0.85);
    ctx.Restore();   
```

We can put all this together into a draw function which also takes the selected user input in order to achieve some highlighting in the canvas.

```fsharp
let drawDartboard (ctx: CanvasRenderingContext2D) (dartToHighlight: Dart) =
    // Init board
    let hightlightColor = "#61bff9"
    resetBoard ctx
    let radius = WIDTH/2. - 10.
    let scoringRadius = (radius * 0.75)

    // Render outer circle
    renderCircle ctx radius "#171918"

    for (index, num) in dartRingNumbers |> List.indexed do
        let doubleColor =
            match dartToHighlight with
            | Double v when num = v -> hightlightColor
            | _ -> if index % 2 = 0 then "#e63322" else "#389536" 
        let tripleColor =
            match dartToHighlight with
            | Triple v when num = v -> hightlightColor
            | _ -> if index % 2 = 0 then "#e63322" else "#389536" 
        let normalColor =
            match dartToHighlight with
            | Single v when num = v -> hightlightColor
            | _ -> if index % 2 = 0 then "#171918" else "#f7e0b6"
        // draw double point ring
        renderSector ctx (scoringRadius) doubleColor index
        // draw normal ring
        renderSector ctx (scoringRadius * 0.953) normalColor index
        // draw triple point ring
        renderSector ctx (scoringRadius * 0.694) tripleColor index
        // draw inner normal ring
        renderSector ctx (scoringRadius * 0.629) normalColor index
        // draw label
        renderLabel ctx num index

    let colorSmallBull = if dartToHighlight = Single 25 then hightlightColor else "#389536"
    let colorBigBull = if dartToHighlight = Double 25 then hightlightColor else "#e63322"
    // draw small bullseye
    renderCircle ctx (scoringRadius * 0.094) colorSmallBull

    // draw bullseye
    renderCircle ctx (scoringRadius * 0.037) colorBigBull
```

The dart function takes the dart value to highlight on the board. If there are none selected, we can pass a Nil into the function, as that is a value that is not represented in the display, therefore the highlight logic would not trigger in that case.

The animation logic, when a combination is selected, is in an infinite loop with the usage of [SetInterval][5] and the step value is for selecting what dart throw we are currently highlighting


Below the canvas, we are also showing the selected combination with the same highlighting in sync with the darts board.

For the user inputs we just have a `numerical input`, a `checkbox` and a `submit` button. Upon pressing the submit button we are initiating the calculating logic. If there are no possible ways to reach the user provided number, we are alerting the user of this problem. If there are results, then we render a select box that is showing all the calculations and upon selecting one of them, we are initiating the highlighting logic mentioned above. Also, the moment a new calculation is initiated or a different result is selected, we are cancelling the previous animation loop.

![Image of the interface](https://i.imgur.com/sU49DBZ.png "Image of the interface")

## Glueing together the pieces

For rendering this into the DOM in the browser, I'm using WebSharper.UI and it's templating engine, with the following html and fsharp code

```html
<div id="main" ws-children-template="Main">
    <div class="holder">
        <div class="form">
            <label for="numberToReach">
                Please provide a number between 1 and 180:
                <input id="numberToReach" type="number" min="1" max="180" ws-var="SelectedNumber" />
            </label>
            <label for="isItCheckout">
                Does it need to be a checkout combination?
                <input id="isItCheckout" type="checkbox" ws-var="IsItCheckout" />
            </label>
            <button ws-onclick="Calculate">Calculate!</button>
            <div>Results:</div>
            <div ws-hole="SelectionBox"></div>
            <div class="selectedDisplay"><span id="dart1">${Selected1}</span><span id="dart2">${Selected2}</span><span id="dart3">${Selected3}</span></div>
        </div>
        <div class="display">
            <canvas ws-attr="CanvasAttr" width="600" height="600"></canvas>
        </div>
    </div>
</div>
```

```fsharp
let selected = Var.Create None
let numberToReach = Var.Create 1
let isItCheckout = Var.Create false
let results = Var.Create [None]
IndexTemplate.Main()
    .CanvasAttr(
        [
            on.viewUpdate selected.View (fun elem currentSelection ->
                animationHandle.View
                |> View.Get (fun handle ->
                    match handle with
                    | None -> ()
                    | Some handle ->
                        JS.ClearInterval handle
                        JS.Document.QuerySelector("span.highlighted") |> fun x -> if x !==. null then x.ClassList.Remove("highlighted")
                        animationHandle.Set None
                )
                let canvasElement = As<CanvasElement> elem
                let context = canvasElement.GetContext("2d")
                highlightDartboard context currentSelection
            )
            on.afterRender (fun elem ->
                let canvasElement = As<CanvasElement> elem
                let context = canvasElement.GetContext("2d")
                drawDartboard context Nil
            )
        ]
    )
    .IsItCheckout(isItCheckout)
    .SelectedNumber(numberToReach)
    .Selected1(selected.View.Map(function None -> "" | Some (d1, _, _) -> Dart.AsDartNotation d1))
    .Selected2(selected.View.Map(function None -> "" | Some (_, d2, _) -> Dart.AsDartNotation d2))
    .Selected3(selected.View.Map(function None -> "" | Some (_, _, d3) -> Dart.AsDartNotation d3))
    .Calculate(fun te ->
        numberToReach.View
        |> View.Get (fun number ->
            isItCheckout.View
            |> View.Get (fun checkout ->
                animationHandle.View
                |> View.Get (fun handle ->
                    match handle with
                    | None -> ()
                    | Some handle ->
                        JS.ClearInterval handle
                        animationHandle.Set None
                )
                let res = getCombinationsFor number checkout
                if res.Length = 0 then
                    JS.Alert <| sprintf "%d cannot be reached with 3 darts! Please try a different number." number
                results.Set (None::res)
                if res.Length > 1 then
                    res |> List.head |> selected.Set 
                else
                    selected.Set None
            )
        )
    )
    .SelectionBox(
        results.View
        |> Doc.BindView (fun options ->
            Doc.InputType.Select [] PrettyPrintDartsSet options selected
        )
    )
    .Doc()
|> Doc.RunById "main"
```

The two most important part of the code are the `Calculate` and the `CanvasAttr` sections.

The `ws-onclick="Calculate"` section in our html lets us bind a click event handler to the function, which in our case will get the current value of the two `ws-var` binded input from our html, to pass it in our `getCombinationsFor` function to generate all the possible combinations shown in the result selector `<select>` element. That `<select>` element is generated from the `View` of the `results`, which is being set in our click handler. The click handler is also responsible for setting the default value of our selected combination: if the number we tried to calculate with cannot be reached with 3 dart throws, then it's setting the `selected` `Var` value to `None`, otherwise it sets it to the first calculated result.

The `CanvasAttr` section is responsible for the animation and the rendering of our canvas. In our html `CanvasAttr` is a `ws-attr` hole, which means we can populate it in F# with a list of `UI.Attr` values, which can take two forms.

- Attributes
- Event handlers

We are utilizing two event handlers for our canvas handling, but keep in mind that these are not standard html event handlers. 

`on.afterRender` takes a function that will be executed immediately after the element we are attaching it to is inserted in the DOM. Therefore we can utilize this layer to render our default state, where nothing is selected.

The `on.viewUpdate` function is a function that takes two argumennts: a `View<'T>` and an a `Dom.Element -> 'T -> unit` function. As the name suggests, the function we are passing in is going to be executed every time the value inside the passed in `View` is changed. Which means we are utilizing this function to handle when the selected combination is changed.

Also, we are calling a slightly altered version of our `drawDartboard` function to handle the animation part.

```fsharp
let highlightDartboard (ctx: CanvasRenderingContext2D) (currentSelection: DartsSet option) =
    match currentSelection with
    | None ->
        drawDartboard ctx Nil
    | Some (d1, d2, d3) ->
        let mutable step = 1
        let highlightAnimation () =
            JS.Document.QuerySelector("span.highlighted") |> fun x -> if x !==. null then x.ClassList.Remove("highlighted")
            let dartToUse =
                if step = 1 then
                    d1
                else if step = 2 then
                    d2
                else
                    d3

            drawDartboard ctx dartToUse
            // highlight matching result field
            JS.Document.GetElementById("dart" + string step).ClassList.Add("highlighted")
            if step = 3 then
                step <- 1
            else
                step <- step + 1

        animationHandle.Set (Some <| JS.SetInterval highlightAnimation 1000 )
```

If our current selection is None, we are doing the same thing as in our `on.afterRender` function. But if a given combination selected we are utilizing `SetInterval` to create the infinite looping logic. The handle that SetInterval returns is stored in a global `Var`, so that our `on.viewUpdate` logic can cancel that loop and restart a new one based on the current selection.

## Closure

I'm happy with how this turned out, but there are always rooms for improvement. I'm pretty sure there are data sets out there to show what is the most used checkout combination for a given number, which could be prioritised at the top of our result set. Or even better, if these are available per professional players, we could have a player selection that would show their most loved combination for a given checkout. But that's for the future :).

Thanks to Sergey Tihon for organizing the [F# Advent Calendar][8] in 2022 again and I'm happy that I could share this nostalgic journey of mine with the community!

[1]: https://en.wikipedia.org/wiki/2023_PDC_World_Darts_Championship
[2]: https://en.wikipedia.org/wiki/Darts#Games
[3]: https://github.com/dotnet-websharper/ui
[4]: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API
[5]: https://developer.mozilla.org/en-US/docs/Web/API/setInterval
[6]: https://github.com/Jooseppi12/fsadvent2022
[7]: https://jooseppi12.github.io/fsadvent2022
[8]: https://sergeytihon.com/2022/10/28/f-advent-calendar-in-english-2022/