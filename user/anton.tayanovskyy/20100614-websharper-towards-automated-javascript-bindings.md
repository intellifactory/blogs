---
title: "WebSharper: Towards Automated JavaScript Bindings"
categories: "f#,websharper"
abstract: "This blog describes the pre-release WebSharper Extensibility Framework."
identity: "2088,74920"
---
This blog describes the pre-release WebSharper Extensibility Framework[^1].

How would I call reasonably complex JavaScript functions from WebSharper? For an example, consider `jQuery.ajax`.

A first attempt would look like this:

```fsharp
[<Inline "jQuery.ajax($settings)">]
let ajax (settings: obj) : obj = null
```

This is hardly acceptable as it throws away all the information present on the documentation page and makes it the user's responsibility to figure out what exactly the `ajax` function expects. But spelling out the precise types requires a lot more work.

In practice, a big part of WebSharper codebase turns out to consist of stubs like these - classes and functions that instruct F# how to call a particular JavaScript API (think jQuery, Ext JS) and provide types for the API. The WebSharper JavaScript-binding code is repetitive, tedious to write and maintain and not straightforward to generate.

At IntelliFactory we have been exprimenting with several approaches to the problem, culminating in something we currently call the Extensibility Framework, or EF for short. The basic idea behind EF is that instead of writing the bindings by hand, we use an F#-embedded DSL to describe them.

`jQuery.ajax` with EF might look like this:

```fsharp
let private AjaxConfig =
    Pattern.Config "AjaxConfig" [] [
        "async" =~ T<bool>
        "beforeSend" =~ J.XMLHttpRequest ^-> T<unit>
        "cache" =~ T<bool>
        "complete" =~ J.XMLHttpRequest * T<string> ^-> T<unit>
        "contentType" =~ T<string>
        "context" =~ T<obj>
        // ..
    ]

    let private JQueryClass =
        Code.Class "JQuery"
        |=> JQuery
        |+> [
             "jQuery.ajax" => AjaxConfig ^-> T<unit>
             "jQuery.ajaxSetup" => AjaxConfig ^-> T<unit>
             "jQuery.contains" => 
                 J.Element?container * J.Node?contained ^-> T<bool>
        // ...
```

The use of the function would then look like this:

```fsharp
jQuery.Ajax(AjaxConfig(Async = false, ...))
```

The good news are several:

 * This is code-generating code - repetitive patterns can be abstracted over with plain functions in F#.

 * If a particular framework provides good documentation, this documentation can be parsed into EF values and then amended by merging it with hand-coded EF values where needed. Bindings generation is then semi-automatic.

 * Some common code-generating patterns will come with EF. Consider `Pattern.Config` above which generates a new configuration object class with the given required and optional fields.

 * Overload generation is automated. In EF, the type for a parameter that accepts either an int or a string can be spelled as `T + T`. The framework then figures out the correct overloads.


There is nothing particularly blog-worthy in the implementation of EF. It is mostly about figuring out the right interface (the most simple and yet diffuclt task there is). I will mention just one implementation detail I am particularly fond of. It is the use of F# operator overloading to describe method and function types. For example, these equalities are made to hold:

```fsharp
T<int -> int -> int> = T<int> ^-> T<int> ^-> T<int>
T<int * int -> int> = T<int> * T<int> ^-> T<int>
```


[^1]: This link is no longer available.
