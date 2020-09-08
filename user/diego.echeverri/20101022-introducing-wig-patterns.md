---
title: "Introducing WIG patterns"
categories: "f#,websharper,dsl,wig"
abstract: ""
identity: "-1,-1"
---
In my last [post](/user/diego.echeverri/20101007-starting-with-websharper-interface-generator.md) you learned how to use WIG to create bindings for 3rd party libraries. We introduced type combinators to express functions and member operators to define properties and methods. Although this is already an improvement over defining the bindings manually, it is not the only way of using WIG.

## Configuration Objects

A common pattern in JavaScript libraries is using configuration objects. In JavaScript it is very common use the object literal syntax. One example may be the [following](https://developers.google.com/maps/documentation/javascript/reference?csw=1#MapOptions)[^1]:

```javascript
var myOptions = {
  zoom: 8,
  center: myLatlng,
  mapTypeId: google.maps.MapTypeId.ROADMAP
};
```

Most of the time, some properties are marked as "Required" (like the `center` property), and some are just optional properties. WIG provides you with a special function to make bindings for such cases.

```fsharp
let MapOptions =
    Pattern.Config "MapOptions"
        {
            Optional =
                [
                    "disableDefaultUI", T<bool>
                    // (...)
                ]
            Required =
                [
                    // The initial Map center. Required.
                    "center", LatLng.Type
                    // (...)
                ]
        }
```

This will generate a new class, where each of the optional members will get translated as getters and setters, and each of the required ones will become parameters in the constructor. This will save you from forgetting to set any of the required attributes, a tangible benefit over using plain JavaScript.

## Enumerations

Another common pattern is using enumeration-like types. Here is an example from the Google Maps API that demonstrates this:

```
visibility | string | Visibility: Valid values: 'on', 'off' or 'simplifed'.
```

In WIG it is possible to generate types for those strings by using the following handy pattern:

```fsharp
let Visibility =
    Pattern.EnumStrings "Visibility" ["on"; "off"; "simplified"]
```

This will avoid tedious debugging sessions to discover any mistyped strings, and instead will provide you with IntelliSense to discover the possible values.

## Building Your Own Patterns

All these patterns have been built using the class construct covered in my last post. This means that is possible to create your own patterns. For example, say you want an Enumeration pattern to cover integers instead of strings. It's possible to write something like:

```fsharp
let EnumInts n l =
    List.map (fun (x, i) -> (x, sprintf "%d" i)) l
    |> List.toSeq
    |> Pattern.EnumInlines n
```

Where `EnumInlines` works in a similar way to `EnumStrings` but receives an arbitrary string that will be inlined. It can be used in the following way:

```fsharp
let MediaError =
    Utils.EnumInts "MediaError" [
        "MEDIA_ERR_ABORTED", 1
        "MEDIA_ERR_NETWORK", 2
        "MEDIA_ERR_DECODE", 3
        "MEDIA_ERR_SRC_NOT_SUPPORTED", 4
    ]
```

## Conclusions

WIG not only provides you with common patterns but it is also extensible. It allows you to define new patterns so you don't have to repeat yourself. This reduces the effort necessary to bind libraries significantly, and also allows you to reuse metabindings across libraries.


[^1]: Link is updated to the latest documentation.
