---
title: "More of Ext JS and WebSharper"
categories: "f#,websharper,extjs"
abstract: "Following up on the previous post on the upcoming WebSharper Ext JS (Native) extension with an example of how to construct a sortable tree widget and reacting to changes of the ordering of tree nodes."
identity: "1910,74742"
---
Following up on the previous [post](/user/joel.bjornson/20100611-writing-ext-js-applications-with-websharper.md) on the upcoming WebSharper Ext JS (Native) extension, here is a new demo containing some live examples[^1]!

Since none of the demos collects any data from the Ext JS controls, here is an example filling the gap. In general, seamlessly being able to create data abstractions and transfer data back and forth between the server and client is a very strong incentive for using the WebSharper extension. The task of today is to define an `Ext.tree.TreePanel` widget for:

 * Sorting hierarchical data.
 * Handling the result of the sorted data.

In order to avoid having to work with `Ext.TreeNodes` directly, we create a simplified data type representing tree structured data. This corresponds to the logical state of the tree widget:

```fsharp
[<JavaScriptType>]
type TreeData = TreeData of list<string * TreeData>
```

Since the tree panel needs to be populated with `Ext.tree.TreeNode` objects we need a function for converting values of `TreeData` to `Ext.tree.TreeNode`:

```fsharp
[<JavaScript>]
let FromTreeData (TreeData data) : Ext.tree.TreeNode =
    // Create a tree node with a unique id.
    let mkNode label =
        Ext.tree.TreeNode (
            Ext.tree.TreeNodeConfiguration(
                id = "Id" + (string <| JMath.Random()),
                text = label
            )
        )        
    // Append tree data to a node
    let rec append (TreeData data) (node : Ext.tree.TreeNode) =
        for (name, tree) in data do               
            let newNode = mkNode name
            append tree newNode
            node.appendChild newNode |> ignore        
    // If list has one element use this a root node
    // otherwise create a new root element.
    match data with
    | (name, tree) :: [] -> 
        let rootNode = mkNode name
        append tree rootNode
        rootNode
    | _                  -> 
        let rootNode = mkNode "Root" 
        append (TreeData data) rootNode
        rootNode 
```

And a corresponding transformer function for getting back a `TreeData` value from a tree node:

```fsharp
[<JavaScript>]
let rec ToTreeData (node: Ext.tree.TreeNode) : TreeData =
    let childTree =         
        node.childNodes
        |> Array.map (fun node ->
            let node = (node :?> Ext.tree.TreeNode)
            let (TreeData data) = ToTreeData node                    
            data
        )
        |> Array.toList                                  
        |> List.concat 
        |> TreeData           
    TreeData [node.text, childTree]
```

A tree widget can now be defined as:

```fsharp
[<JavaScript>]
let SortableTree (treeData : TreeData) (han: TreeData -> unit) =
    // Create a tree and assign it the root node.   
    let tree =                      
        Ext.tree.TreePanel(                        
            Ext.tree.TreePanelConfiguration(
                title = "Tree Panel",
                width = 250.,
                useArrows = true,
                animate = true,
                enableDD = true,
                root = FromTreeData (GetTreeData ())
            )
        ) 
    // Handling changes
    tree.onDragdrop(fun panel node _ _ ->
        ToTreeData node
        |> han           
    )
    tree 
```

Along with an initial value it accepts a function to be called with the current tree data value every time the user reorders the nodes.

Here is an example of how to use the `SortableTree` function:

```fsharp
[<JavaScript>]
let Main () =
    let tree = 
        SortableTree (DataStore.GetTreeData ()) (fun treeData ->
            DataStore.Update treeData
            ()
        )
    Div []
    |>! OnAfterRender (fun el ->
        tree.render(el.Dom)
    )
```

And the resulting tree panel with drag and drop support for ordering nodes:

![](/assets/145.png)



[^1]: This link is no longer available. A (broken) snapshot of the page from the Internet archives is available [here](https://web.archive.org/web/20101123072954/http://intellifactory.com/products/wsp/samples/ExtJs.aspx).

