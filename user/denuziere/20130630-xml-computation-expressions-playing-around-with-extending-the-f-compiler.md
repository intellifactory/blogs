---
title: "XML computation expressions: Playing around with extending the F# compiler"
categories: "edsl,ce,dsl,compiler,f#"
abstract: "Existing embedded DSLs for XML in F# all have their quirks and inconvenients. I'm presenting a small patch to the F# compiler that allows to implement a better XML EDSL."
identity: "3394,76694"
---
It's been a couple months since [this discussion](https://groups.google.com/forum/?fromgroups=#!searchin/fsharp-opensource/xml/fsharp-opensource/lvqnyb1CZUA) in the F# Open-Source mailing list about a unified F# EDSL for XML trees, but I've remained intrigued by this problem and wanting to improve on the solutions proposed. I found [Tomas Petricek's computation expression-based solution](http://fssnip.net/hf), in particular, quite nice in its conciseness and non-reliance on custom operators, which makes it much more intuitive. Its big problem, though, is its complete reliance on side-effects at every level.

## So much impurity...

Now, I'm not saying that such an EDSL *must* be completely pure in order to be even considered. After all, WebSharper's own HTML EDSL relies on side-effects. But at a much smaller scale: for example, in the following code, the only side-effect is done by the <c>(-<)</c> operator, which destructively adds children to a node.

```fsharp
let ``WebSharper node`` =
    Div [Class "header"; Id "main-header"] -< [
        Span [Text "This is the contents of the header."]
        Br []
        Span [Class "clear"]
    ]
```

On the other hand, Tomas's EDSL uses a global variable `h` containing an internal pointer to the current position in the XML tree where the next node will be written. Which means that not only does every single operation (moving up or down in the tree, inserting a node or an attribute) relies on mutation, but a wrong use of a method of `h` can land you in the wrong place of the tree, potentially throwing an error when closing the computation expression and trying to move up from the root. Ugh... Surely we can do better!

## ... but at what cost?

Well, as it turns out, we can't.

At least, not if we want to keep a clean and concise syntax. We could, of course, make use of the `yield` and `yield!` keywords, or even [custom operations](http://msdn.microsoft.com/en-us/library/hh289709.aspx) if we find these keywords too long.

```fsharp
let ``Pure CE, first attempt`` =
    // "attr", "node" and "nodes" are custom operations here
    div {
        attr ("class", "header")
        attr ("id", "main-header")
        node (span { text "This is the contents of the header." })
        nodes [
            br
            span { attr ("class", "clear") }
        ]
    }
```

... And the result is more cluttered than any of the existing EDSLs. Not good.

Actually, there is a simple reason why a Computation Expression-based XML EDSL needs to be destructive if it must be concise.

For those unfamiliar with the way F# Computation Expressions are implemented, this is how it works: the keyword that you place at the beginning of a CE (`async`, `seq`, etc — or in the above code, `div`) is actually an object, called the builder. Each keyword inside the CE is translated to a method call on the builder according to a set of [simple desugaring rules](http://msdn.microsoft.com/en-us/library/dd233182.aspx).

According to these rules, when a bare expression is met within a CE (`{| other-expr; cexpr |}`), it is transformed into a sequence of expressions (`other-expr; {| cexpr |}`). Which means its return value is discarded, and the only useful thing it can do is a side-effect. No way out of this: if we want to use bare expressions, they must be impure.

## Extending F# for fun, we'll see about profit

At this point, I chose to leave the realm of what can be done, and to enter the realm of what could be done. After all, you never do anything new if you don't start asking "What if?"

What if the bare expression expansion rule was different? What if instead of simply transforming it into a sequence of expressions, it called a new method on the CE, let's call it `Sequence`, that would perform whatever transformation we want it to?

Well, there's only one way to find out. Off to [the compiler](https://github.com/fsharp/fsharp) we go!

This being my first foray into the F# compiler source, it took me some time to find my way around. But fortunately, the change needed to implement the `Sequence` method ended up being [even smaller](https://github.com/Tarmil/fsharp/commit/fca70beefb47d005f198eeca3bab1ab7b1d138ef) than I thought it would be. The first chunk of the diff implements the following transformation:

```fsharp
{| other-expr; cexpr |} --> builder.Sequence(other-expr, {| cexpr |})
```

and the second chunk implements:

```fsharp
{| other-expr |} --> builder.Sequence(other-expr, builder.Zero())
```

The nice part about this is, just like the existing `Run` and `Delay` methods, if `Sequence` is not implemented by the builder, then we can just revert to the default behavior, making this change 99% retro-compatible. The only breaking case is if an existing builder happens to have a `Sequence` method for its own purposes, then this method will be mistaken for an implementation of this feature.

## A better XML than XML

So, back to our XML EDSL business, but now with our custom F# compiler. In the F# Open-Source mailing list thread, Anton Tayanovskyy listed the different cases that our EDSL should be able to concisely represent.

```xml
<!-- empty element with no attributes -->
<hr />

<!-- empty element with attributes -->
<link rel="stylesheet" href="foo.css" />

<!-- empty element with text-only content -->
<title>foo</title>

<!-- empty element with attributes and text-only content -->
<div id="main">foo</div>

<!-- element with no attributes and element-only content -->
<div><hr/></div>

<!-- element with attributes and element-only content -->
<div id="main"><hr/></div>

<!-- element (possibly with attributes) with mixed content -->
<div id="address">Foo St<br/>Bar Town</div>
```

With our newly implemented CE syntax extension, this is how we'll be able to represent them:

```fsharp
// empty element with no attributes
HR

// empty element with attributes
Link { Rel "stylesheet"; HRef "foo.css" }

// empty element with text-only content
Title { "foo" }

// empty element with attributes and text-only content
Div { Id "main" } { "foo" }

// element with no attributes and element-only content
Div { HR }

// element with attributes and element-only content
Div { Id "main" } { HR }

// element (possibly with attributes) with mixed content
Div { Id "address" } { "Foo St"; Br; "Bar Town" }
```

Looks pretty cool, doesn't it? [Here is the implementation.](https://gist.github.com/Tarmil/5894620) It actually relies on a final couple of tricks, but unlike the previous one, these tricks are entirely legit F# :)

### The builder

The first trick is simple: the node type is the builder! This allows us to have both

```fsharp
Div : Node
```

and

```fsharp
Div { HR } : Node
```

Even better, if we have an existing node `myNode`, then we can add children to it with a CE:

```fsharp
myNode { Div }
```

### Attributes

The value carried around by the CE is actually the list of children that will be added to the node. The final creation of the node is done by the `Run` method, which gets called automatically by the CE desugaring (as shown [here](http://msdn.microsoft.com/en-us/library/dd233182.aspx)).

But what about attributes? One thing we could do is, instead of carrying a list of children, carry a pair: a list of children and a list of attributes. But this would allow mixing children and attributes within the same CE:

```fsharp
Div { Id "main"; Span { "contents" } }
```

and I'm not very fond of the idea: it contradicts this fundamental distinction, in XML, between attributes and child nodes. So I decided instead to take advantage of method overloading. I have two versions of `Sequence`:

```fsharp
type Node =
    // ...
    member this.Sequence (n: Node, l: Node list) = n :: l
    member this.Sequence (a: Attribute, l: Attribute list) = a :: l
```

It is now impossible to mix nodes and attributes, because there is no method with arguments `Node * Attribute list` or vice-versa.

In the end, we have what could be considered two different CEs using the same builder; one for child nodes, and one for attributes. And since the type finally returned by the CE is the builder type, we can chain them:

```fsharp
// Using the child node CE
Div { Span }
// Using the attribute CE
Div { Id "main" }
// Chaining both CEs
Div { Id "main" } { Span }
```

### Sequences of children

It is very common, when building HTML, to need to insert a sequence of children into a parent node. Rows into a table, articles into a page, etc. This is actually, in my opinion, the main weakness of WebSharper's EDSL. It is certainly possible to insert a sequence of children, but not very practical:

```fsharp
let childrenToAdd = [Div [Text "c1"]; Div [Text "c2"]]

/// The parent as it would be without the children.
let parent = Div [
    H1 [Text "Title"]
    // We want to put the children here
    Div [Class "footer"]
]

/// First possibility: use yield!, forcing us to add yield to all other children.
let parent' = Div [
    yield H1 [Text "Title"]
    yield! childrenToAdd
    yield Div [Class "footer"]
]

/// Second possibility: use (-<) to break down the children into several sequences.
let parent'' = Div [
    H1 [Text "Title"]
] -< childrenToAdd -< [
    Div [Class "footer"]
]
```

Fortunately, our new EDSL can do this better, in two different ways even. The first one is to (ab)use method overloading again:

```fsharp
type Node =
    // ...
    member this.Sequence(ns: seq<Node>, l: Node list) = List.ofSeq ns @ l

let childrenToAdd = [ Div { "c1" }; Div { "c2" } ]

let parent = Div {
    H1 { "title" }
    childrenToAdd
    Div { Class "footer" }
}
```

The second one is to implement the `For` method of the CE builder. This allows us to insert for loops right into our XML!

```fsharp
let parent = Div {
    H1 { "title" }
    for c = 1 to 2 do
        Div { "c" + string i }
    Div { Class "footer" }
}
```

## In conclusion

Obviously, the EDSL presented here is only a toy and a yearning for what could be; and it will remain so as long as it depends on a custom patched compiler. This being said, with the recent news that the F# team intends to put out [more frequent updates to the language](http://blogs.msdn.com/b/fsharpteam/archive/2013/06/27/announcing-a-pre-release-of-f-3-1-and-the-visual-f-tools-in-visual-studio-2013.aspx), one could imagine a feature similar to `Sequence` making its way into the official F# compiler some day. Hey, don't blame me for hoping!
