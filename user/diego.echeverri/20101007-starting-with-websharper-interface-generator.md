---
title: "Starting with WebSharper Interface Generator"
categories: "f#,websharper,wig"
abstract: "WIG is an F# DSL and source-level translator for creating bindings for 3rd party libraries."
identity: "-1,-1"
---
The fact is that there is already plenty of awesome JavaScript libraries. In the spirit of reusability, WebSharper allows you to create wraps around third party libraries just as is shown in a [previous blog entry](/user/granicz/20100930-tutorial-implementing-a-shopping-cart-with-websharper.md). Sometimes though, the library you want to create wraps for contains hundreds of classes, configuration objects and enumerations. Doing this manually is not only error-prone but extremely boring. For this reason we created the WebSharper Interface Generator (WIG)), soon to be available a standalone tool to complement the WebSharper 2.0 toolset. WIG is an F# DSL and source-level translator for creating bindings for 3rd party libraries. WIG provides you with:

 * Common patterns in wrap creation (Enumerations, Configuration objects)
 * Concise syntax (There's 1/5-1/8 ratio between the generated code and the WIG code)
 * Programmatic code generation (Generate code directly from the documentation)
 * No back-reference problems that you may encounter doing the bindings manually.

## A Simple Example: HTML5 WebStorage Bindings

One of the proposed features for HTML5 is an [API for persistent data of key-value pair data in Web clients](https://html.spec.whatwg.org/multipage/webstorage.html)[^1]. The main interface in the current draft is the following:

```
    interface Storage {
      readonly attribute unsigned long length;
      getter DOMString key(in unsigned long index);
      getter any getItem(in DOMString key);
      setter creator void setItem(in DOMString key, in any value);
      deleter void removeItem(in DOMString key);
      void clear();
    };
```

The instance implementing this interface is available through `window.localStorage`.

## Defining the class

Let's check how the code for it would look like in WIG:

```fsharp
    module LocalStorage =
        let Storage =
            let Storage = Class "Storage"
            Storage
            |+> ["localStorage" =? Storage |> WithGetterInline "localStorage"]
            |+> Protocol [
                "length"     =? T<int>
                "key"        => T<int> ^-> T<string>
                "getItem"    => T<string> ^-> T<string>
                "setItem"    => T<string> * T<obj> ^-> T<unit>
                "removeItem" => T<string> ^-> T<unit>
                "clear"      => T<unit> ^-> T<unit>
            ]
```

The first thing you do is creating an empty class called `Storage`. The name you give to that class is the name that will be used in the generated code. In this case this name is not really important since this class doesn't have any public constructor. The first thing you'll notice is the operator `|+>`. What it does is that it changes the current class and adds the given members. Its first usage provides the class with a static getter for the `localStorage` object. The `=?` operator creates a getter with the given name and the given type. You can override its translation by using `WithGetterInline`. The second part uses `Protocol` for creating a list of instance members that will be added to the object. Method members are created by using the `=>` operator. The types of the method members can be written by using the already existing types.

 * Product types (i.e tuples) can be written using the `*` operator
 * Function types can be written by using the `^->` operator
 * Already existing types can be used with the `T` function.
 * WIG also supports operators to create overloads, optional arguments and multi-arg functions.


## Adding the class to a Namespace

WIG provides you with the tools to select how you will organize the generated code.

```fsharp
    module Main =
        let Assembly =
            Assembly [
                Namespace "IntelliFactory.WebSharper.HTML5" [
                     LocalStorage.Storage
                ]
            ]
        do Compiler.Compile stdout Assembly
```

First you create a single `Assembly` with a single `Namespace`. In this `Namespace` you'll have the `Storage` class. Having that defined is a matter of "compiling" the `Assembly` to generate the code.


## Using the generated code

Having generated and compiled the code. It is possible to reference the dll in any of your WebSharper projects. Let's see how a simple Storage example looks like:

```fsharp
    [<JavaScriptAttribute>]
    let LocalStorage () =
        Div [
            H1 [Text "LocalStorage Test"]
        ]
        |>! OnAfterRender (fun elem ->
            let storage = Storage.LocalStorage
            let key = "intReference"
            let intReference = storage.GetItem(key)
            if intReference = null || intReference = Undefined then
                storage.SetItem(key, "0")
            else
                let oldValue = int (intReference)
                storage.SetItem(key, (oldValue + 1).ToString())
            elem.Html <- elem.Html
                         + "<h2>localStorage is now: ("
                         + storage.GetItem(key) + ")</h2>")
```

The interesting part starts in the line #7. You access the members of the class just like any other WebSharper class. One of the cool things that WIG does for you is changing the member names in order to comply with .NET naming conventions.


## Conclusions

This is a small real-world example of how WIG helps you to simplify the generation of stubs for 3th party libraries. Keep in mind that this was just a simple example. Given enough time it would be possible to do things like:

 * Create an IDL parser to do part of the dirty work for you.
 * Scrape the WebStorage draft website to check whether the current bindings are up-to-date.


[^1]: Link updated to the latest version.
