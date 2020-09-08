---
title: "Interfacing with dynamic JavaScript APIs in WebSharper"
categories: "f#,websharper"
abstract: "This article gives and example of how to interface with dynamically typed JavaScript APIs."
identity: "1906,74738"
---
One of our visions with WebSharper is to enable you to do do all aspects of client side web development in F#. First and foremost by translating F# code into efficient JavaScript code, but also by providing bindings (and the ability to create new bindings) to a variety of popular JavaScript libraries such as jQuery, GoogleMaps and O3D.

Consider for example the function `addClass` from jQuery. The WebSharper API contains two overloaded versions representing the different ways of invoking the method. For example:

```javascript
JQuery.Of("#myTable").AddClass("newClass")

JQuery.Of("#myTable").AddClass(fun (ix, _) ->
    if ix % 2 = 0 then
        "evenRow"
    else
        "oddRow"
)
```

The typedness of the F# API effectivly restricts the user from invoking methods in an invalid way. The following examples result in a compilation error:

```fsharp
JQuery.Of("#myTable").AddClass(33)
JQuery.Of("#myTable").AddClass(fun _ -> ())
```

However, there are some dynamic features in some JavaScript libraries that are not straight forwardly mapped to a typed setting.

As an example imagine a JavaScript method `setCss` that only accepts its arguments as an object of name/value pairs for setting CSS properties. In JavaScript you would use the method as in:

```javascript
node.setCss({
    background-color : "#CCC",
    padding : 10
});
```

But how would you map this to F#? Unless constructing a configuration type that enlists all CSS properties, you have to fall back to using `obj` as the type of the argument to the function.

Invoking the function from F# is no longer as straightforward as the corresponding JavaScript code, and typically require you to define additional types covering the different ways in which you are using the function.

The example from above can be written in F# as:

```fsharp
type CssProps =
    {
        background-color : string
        padding : int
    }

node.SetCss
    {
        background-color = "#CCC"
        padding = 10
    }
```

The additional boilerplate code for defining types can certainly be annoying. For this purpose, it may be convenient to mimic the ability to create anonymous types (as achieved by the record syntax in JavaScript).

Fortunately, this is quite possible using a few utility functions:

```fsharp
[<JavaScript>]
let (:=) x y : (string * obj) = (x, y)

[<Inline "void($r[$k]=$v)">]
let Set (r: obj) (k: string) (v: obj) : unit = ()

[<JavaScript>]
let (!) (fields: seq<(string * obj)>) =
    let r = obj ()
    for (k,v) in fields do
        Set r k v
    r
```

The `(!)` operator accepts a sequence of name/value pairs and returns a JavaScript object populated with the corresponding fields.

The same example can now be written as:

```fsharp
SetCss ![
    "background-color" := "#CCC"
    "padding" := 10
]
```

This is obviously not type-safe, but at least it enables you to interface with dynamic aspects of JavaScript with very little syntactic overhead. A feature for creating anonymous objects (like in the example above) will be included in the next release of WebSharper.
