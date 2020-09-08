---
title: "Writing Ext JS applications with WebSharper"
categories: "f#,websharper,extjs"
abstract: "Some examples of how to use the upcoming WebSharper binding for Ext JS."
identity: "1912,74744"
---
[Ext JS](https://www.extjs.com/) is a versatile JavaScript framework for creating advanced desktop-like web applications. It provides a rich set of widgets and utility functions for constructing panels, layout managers, forms etc. Basically, it contains everything you would expect from a GUI toolkit along with a well documented and consistent API.

Naturally this is something we would love to have support for in WebSharper and recently we have been working on creating a binding to the Ext JS framework.

Our intention is to support the full extent of Ext JS while adding the extra level of type safety provided by F#.

## A few examples

The following example shows how you can create a button that alerts a message when clicked using the `Ext.MessageBox` class. At first a version written in plain JavaScript:

```javascript
var button = new Ext.Button(); 
button.setText("Click" );
button.addListener("click",  function (x, y) { 
    Ext.MessageBox.alert("Alert", "Button clicked")
});
```
> **Editorial note** <br />
> This resource needs to be re-added.
> ![](https://intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=137)

Instead, using WebSharper the code could be written as:

```fsharp
let button = Ext.Button()
button.setText "Click Me!" |> ignore
button.onClick(fun _ _->
    Ext.MessageBox.alert("Alert", "Button clicked", ignore)
    |> ignore        
)
```

Ext JS widgets are customized by passing a configuration object to the constructor function. Here is another example of how to create a button and specifying the `text` and `width` properties:

```javascript
var button = 
    new Ext.Button(
    {
        text : 'Click Me!',
        width : 100,  
    }
);
```

In the WebSharper extension, such configurations are given their own types. For example, the `Ext.Button` constructor accepts an argument of type `Ext.ButtonConfiguration`.

```fsharp
let button =
    Ext.Button (
        Ext.ButtonConfiguration(
            text = "Click",
            width = 100.
        )
    )
```

As you can see even the syntax between the JavaScript and the F# versions of the examples are quite similar. Nevertheless, there are some good points for using the WebSharper extension. First and foremost, you get static type checking and IntelliSense support for looking up methods and properties. For example, keeping track of all the different configuration settings is hard for such a large library as Ext JS, thus static typing and code completion is helpful. Here is an example of IntelliSense in play:

> **Editorial note** <br />
> This resource needs to be re-added.
> ![](https://intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=138)

Another advantage of using the WebSharper binding rather than working with JavaScript directly is the ability to use all the powerful F# constructs and intermixing with other WebSharper functionality. Especially the seamless interaction with server side functionality via RPC method calls is particularly convenient.

In the following example an `Ext.TabPanel` is constructed for displaying three elements on separate tabs. Note that you can use the WebSharper HTML constructor functions for adding tab content (all it takes is to convert them into an `Ext.Element` object using the `ExtT.get` function).

```fsharp
[<JavaScript>]
let TabsSample () =
    let tabs =
        [|
            "Tab 1", H1 [Text "Tab 1 - Content"]
            "Tab 2", B [Text "Tab 2 - Content"]
            "Tab 3", I [Text "Tab 3 - Content"]
        |]
        |> Array.map (fun (title,content) ->
            Ext.PanelConfiguration (
                title = title,
                padding = 10,
                items = [| 
                    Ext.Container(
                        el = ExtT.get(content.Dom)
                    )                            
                |]
            )
        )        
    Ext.TabPanel(
        Ext.TabPanelConfiguration(
            height = 100.,
            width = 300.,
            activeTab = 0, 
            animScroll = true,  
            items = tabs
        )
    )
```

> **Review note** <br />
> This resource needs to be re-added.
> ![](https://intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=139)

To further illustrate the use of ExtJS and the provided widgets, below is a small sample of how to construct an `Ext.tree.TreePanel` and populate it with nodes:

```fsharp
[<JavaScript>]
let TreeSample () =
    // Function for creating a tree node.
    let mkNode label =
        Ext.tree.TreeNode (
            Ext.tree.TreeNodeConfiguration(
                id = "Id" + (string <| JMath.Random()),
                text = label
            )
        )                                
    // Create the root node.
    let rootNode = mkNode "Root"

    // Populate root node.
    for i = 1 to 10 do
        let node = mkNode ("Node " + (string i))
        rootNode.appendChild(node) |> ignore
        
        // Attach some sub nodes
        let numSubNodes = int (JMath.Random() * 5.)
        for j = 1 to numSubNodes do
            mkNode ("Sub Node " + string i)
            |> node.appendChild
            |> ignore

    // Create a tree and assign it the root node.             
    Ext.tree.TreePanel(                        
        Ext.tree.TreePanelConfiguration(
            title = "Tree Panel",
            width = 250.,
            useArrows = true,
            animate = true,
            enableDD = true,
            root = rootNode
        )
    )
```

Simply by changing some configuration settings in the `Ext.TreePanelConfiguration` object you can allow dragging and dropping nodes, adding animations etc.

> **Review note** <br />
> This resource needs to be re-added.
> ![](https://intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=140)

The Ext JS extension will be available for download shortly!
