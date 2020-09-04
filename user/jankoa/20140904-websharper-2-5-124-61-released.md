---
title: "WebSharper 2.5.124.61 released"
categories: "f#,websharper"
abstract: "This new version brings bugfixes and WIG improvements"
identity: "4015,77380"
---
## Bugfixes

 * Fixes to `Math.atan2`, `String.Replace`, `Observable.pairwise` and `Observable.scan` proxies.
 * Observable functions propagate exceptions originating in mapping/folding functions properly.
 * ParamArray in a constructor by WIG translates correctly.
 * WIG now correctly sets the full names of generic types so that the same type name with different number of type arguments can be used.


## Possible breaking changes

Bounds checks are introduced on Array functions to match .NET behavior so they no longer return undefined values which could lead to hard to find bugs. This breaks previous code which uses the Array type as a sparse array. For using JavaScript operations on Arrays, casting to `EcmaScript.Array` or using the `.ToEcma()` extension method is recommended.


## Additions to WebSharper Interface Generator

* Support for indexed properties. Examples:

    ```fsharp
    "item" =@ T<obj> |> Indexed T<int> // has inline "$this[$index]"
    "prop" =@ T<obj> |> Indexed T<int> // has inline "$this.prop[$index]"
    ```

    A property named "item" is handled specially for creating the default inline, the object itself gets indexed.

* Support for type constraints. Example:

    ```fsharp
    Generic - fun b -> "test" => (b |> WithConstraint [ T<System.Collections.IEnumerable> ]) ^-> T<unit>
    ```

    The `WithConstraint` function destructively sets the constraints on the type parameter. Use it anywhere but only once inside the function passed to the Generic helper.
