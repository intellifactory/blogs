---
title: "Fun with dependent formlets"
categories: "f#,websharper,formlets"
abstract: "In this post I'm implementing a simple spread-sheet grid widget with dynamic summation of rows and columns, using dependent formlets."
identity: "1909,74741"
---
To give some hints of what you can do with the WebSharper formlet library, the objective of the following exercise is to create a simple spread-sheet like widget based on formlets.

In short, we want to create a grid for inputting integer values and add a dynamically updated column displaying the sums of all rows along with a row for displaying the sums of all columns.

At first we define a general combinator for constructing a generic grid formlet:

```fsharp
[<JavaScript>]
let Grid rows cols formlet =
    List.init rows (fun _ ->
        List.init cols (fun _ ->
            formlet
        )
        |> Formlet.Sequence
        |> Formlet.Horizontal
    )
    |> Formlet.Sequence
```

The function accepts two parameters for specifying the number of rows and columns and an inner formlet constructor. The resulting formlet produces values of type list of list of `T`, where `T` is the value type of the inner formlet. For the spread-sheet widget we want to be able to input integer values, why the following helper function comes in handy:

```fsharp
[<JavaScript>]
let IntField (n: option<int>) =
    let initVal =
        match n with
        | Some v -> string v
        | None -> ""
    Controls.Input initVal
    |> Validator.IsInt "Only integer values allowed"
    |> Enhance.WithValidationFrame
    |> Formlet.MapResult (fun res ->
        match res with
        | Success vl -> int vl
        | _          -> 0
        |> Success
    )
```

The `IntField` is just a `Controls.Input` with integer validation mapped with a function for translating non integer values to 0. In this way, forms instantiated using `IntField` will never enter a failing state. Here is an example of a grid with 7 rows and 4 columns used with `IntField` as the inner formlet constructor:

```fsharp
Grid 7 4 (IntField None)
```

![](/assets/142.png)

Next step is to enhance the grid formlet with an extra column for displaying the summed values of each row. The following combinator creates a dependent formlet, holding the extra grid column:

```fsharp
[<JavaScript>]
let WithRowSums grid  =
    // Composing elements horizontally
    let horizontal (e1 : Element) (e2 : Element) =
        Table [TBody [TR [TD [e1]; TD [e2]]]]
    // Creates a dependent formlet containing the row sums
    Formlet.BindWith horizontal grid (fun rows ->
        // Sum each row
        rows
        |> List.map (fun row ->
            Some (List.sum row)
            |> IntField
        )
        |> Formlet.Sequence
        |> Formlet.Map (fun rowSums ->
            List.zip rows rowSums
            |> List.map (fun (cells,sum) -> cells @ [sum])
        )
        |> Formlet.MapElement (fun el ->
            // Wrap with a css class
            Span [Class "sum"] -< [el]
        )
    )
```

To lay out the two form components (the grid and the extra column) horizontally, `Formlet.BindWith` applied with a function (`horizontal`) for specifying the composition is used.

In a similar fashion a new combinator, enhancing the grid with an extra row containing the column sums may be defined. This time the default formlet builder is utilized since it will compose the visual elements vertically and we want to display the additional row beneath the grid:

```fsharp
[<JavaScript>]
let WithColumnSums grid  =
    // The grid enhanced with an extra sum row
    Formlet.Do {
        let! rows = grid
        return!
            // Sum each column
            rows
            |> List.reduce (fun sum row ->
                List.zip sum row
                |> List.map (fun (x,y) -> x + y)
            )
            |> List.map (Some >> IntField)
            |> Formlet.Sequence
            |> Formlet.Map (fun row -> rows @ [row])
            |> Formlet.Horizontal
            |> Formlet.MapElement (fun el ->
                Span [Class "sum"] -< [el]
            )
    }
```

The only thing remaining now is to compose the previously defined combinators into a nice grid enhanced with row and column sums:

```fsharp
[<JavaScript>]
let GridWithColumnAndRowSums () =
    IntField None
    |> Grid 7 4 
    |> WithColumnSum
    |> WithRowSum
```

Here is an example of `GridWithColumnAndRowSums` in action:

![](/assets/141.png)

Note that every time a value in the grid is changed the dependent cells containing the summed values are updated immediately.
