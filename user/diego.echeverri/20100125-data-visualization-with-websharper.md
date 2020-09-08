---
title: "Introducing WIG patterns"
categories: "f#,websharper,web,googlevisualization"
abstract: "In this post I'll show you how to make a nice visualization similar to the ones used by Hans Rosling. For this, we'll use the WebSharper Google Visualization bindings available as an extension package to the core platform."
identity: "-1,-1"
---
Some time ago, I was looking through some of TED videos and I stumbled upon a really nice presentation by Hans Rosling about AIDS. After that initial video I went through [all his videos](https://www.ted.com/search?q=Hans+Rosling). It's surprising how much difference it makes to have the right visualization.

In this post I'll show you how to make a nice visualization similar to the ones used by Hans Rosling. For this, we'll use the WebSharper Google Visualization bindings available as an extension package to the core platform.

## Getting the data

For this example, I'll plot some population information taken from the population division of the UN[^1]. And to make things easy, I edited the information as an array of tuples. In principle, you could just use any source for the data. For example you could call the database asynchronously, use an Xml file or a web-service. It doesn't really matter. Just to illustrate how my hard-coded data looks like:

```fsharp
let arrayData = [|
    ("Colombia",1950,12000)
    ("Colombia",1955,13828)
    ("Colombia",1960,16006)
    ...
    ("Ukraine",1995,51063)
    ("Ukraine",2000,48870)
    ("Ukraine",2005,46936) |]
```

We now have to wrap the data inside a `DataTable` object. This one is defined inside the `IntelliFactory.WebSharper.Google.Visualization.Base` module. Keeping in mind that these are just bindings for the Google Visualization API, we should check [the documentation](https://developers.google.com/chart/interactive/docs/gallery/motionchart?csw=1)[^2] to see what's the expected format.

 * The first column must be of type `string` and contain the entity names (e.g., `"Apples"`, `"Oranges"`, `"Bananas"` in the example above).

 * The second column must contain time values. Time can be expressed in any of the following formats:

    ```
    * Year - Column type: 'number'. Example: 2008.
    ...
    ```

 * Subsequent columns can be of type `number` or `string`. Number columns will show up in the dropdown menus for X, Y, Color and Size axes. String columns will only appear in the dropdown menu for Color.

## Wrapping the data inside the DataTable

With the previous information about the format expected by the `MotionChart` we can proceed to add the columns:

```fsharp
    let data = new Base.DataTable()
    data.addColumn("string", "Country") |> ignore
    data.addColumn("number", "Year")      |> ignore
    data.addColumn("number", "Population")   |> ignore
```

To add the data we just iterate through the collection with the `addRow` method. The order of every cell should correspond to the order on which we declared the columns.

```fsharp
    let V: obj -> Cell = Cell.Value
    for (c, y, p) in arrayData do
        data.addRow [|V c; V y; V p|] |> ignore
```

## Creating the Visualization

The following function creates the visualization:

```fsharp
let MotionChart () =
    Div []
    |> On Events.Attach (fun container _ ->
        let visualization = new Visualizations.MotionChart(container)
        let options = {Visualizations.MotionChartOptions.Default 
                            with width = 600.
                                    height = 300.}
        visualization.draw(MotionChartData, options))
```

There are a couple of things to remember:

 * First, it is necessary to create the visualization using the `Attach` event. The reason is that some of libraries that we rely on, need that the element is already attached to the DOM.

 * Every visualization receives a configuration object. If you check the API's documentation you'll notice that they use simple object literals. In our case we created record types to simulate those options. This gives us the object literal syntax and strong typing. All configuration options are called `(VisualizationName)Options`.

You can isolate the element and create an ASP.NET control to display the visualization. For example:

```fsharp
[<JavaScriptType>]
type BlogMotionChart() =
    inherit Web.Control()

    [<JavaScript>]
    override this.Body = Samples.BlogMotionChart()
```

## Playing with the Visualization

The cool thing about the visualization that I chose to show is that it contains different "Modes". The first mode is the one that Hans Rosling uses:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=125)

This one allows you to see the behavior of three variables through time (x, y axis + size of the sphere). In our case it isn't really helpful because we just have the population variable.

The second visualization uses a histogram. This one is a bit more helpful in our case:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=126)

And finally a simple line diagram from the data:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=127)

## Conclusions

We can start to throw hypotheses about the behaviour of our data. For example: What happened to Ukraine in the nineties? There's a big drop in the population. Was it related to the [hyperinflationary levels](https://en.wikipedia.org/wiki/Ukraine#Economy)? According to [this page](https://jamestown.org/program/census-ukraine-more-ukrainian/):

"The acute socioeconomic crisis of the 1990s discouraged families from having children, especially in urban areas, something that affected Russians and Russian speakers more than others."

I really don't have any idea if there was really a relation between the two variables but it sounds plausible. The important thing here is how easy it was to make this visualization. There are a lot of possibilities in business intelligence, scientific visualizations, creating reports... you name it. Go on and give it a try!


[^1]: Original link is dead and was removed.

[^2]: Link is pointing to the latest version.
