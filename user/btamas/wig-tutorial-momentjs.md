---
title: "Creating a WebSharper binding for Moment.js"
categories: "f#,wig,websharper"
abstract: ""
---

# Creating a WebSharper binding for `Moment.js`

Hello!
In this blog I will show you how to create a WebSharper extension using WebSharper Interface Generator (WIG). For this tutorial I'm going to be binding `Moment.js`, a date parsing/formatting library.

## First steps

Let's create a WebSharper Extension project first. Make sure you have `WebSharper.Templates` installed, which you can do with:

```
dotnet new -i WebSharper.Templates
```

Using CLI:

```
dotnet new websharper-ext -o WebSharper.Moment
```

This made us a template project with the following files:

* Main.fs
* WebSharper.Moment.fsproj
* wsconfig.json

For now, we work in `Main.fs`. As you can see too if you follow along, it created a template file where many WIG functionality is shown. For information about WIG, please visit the [documentation](https://developers.websharper.com/docs/v4.x/fs/wig). Now let's clean the file a bit:

```fs
namespace WebSharper.Moment

open WebSharper
open WebSharper.JavaScript
open WebSharper.InterfaceGenerator

module Definition =

    let Assembly =
        Assembly [
            Namespace "WebSharper.Moment" [

            ]
        ]

[<Sealed>]
type Extension() =
    interface IExtension with
        member ext.Assembly =
            Definition.Assembly

[<assembly: Extension(typeof<Extension>)>]
do ()

```

Nice! Let's get on with the binding now!

## Binding

We should get the required resources first. The `cdnjs` for Moment can be found [here](https://cdnjs.com/libraries/moment.js). We usually put this in the assemblies as `Resource` which will add a `script` tag to the html's head with this as `src`. Now in a regular case you would do this:

```fsharp
Assembly [
    Namespace "WebSharper.Moment.Resources" [
        Resource "Js" "https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.29.1/moment.min.js"
    ]
    Namespace "WebSharper.Moment" [

    ]
]
```

But since we will do the binding for `Moment Timezone` too ([Moment Timezone](https://momentjs.com/timezone/docs/)) which requires the default Moment.js. We do something like this:

```fsharp
module Res =
    let Js =
        Resource "Js" "https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.29.1/moment.min.js"

    let TzJs =
        Resource "TimezoneJs" "https://cdnjs.cloudflare.com/ajax/libs/moment-timezone/0.5.33/moment-timezone.min.js"
        |> Requires [Js]

module Definition =
    ...

    let Assembly =
        Assembly [
            Namespace "WebSharper.Moment.Resources" [
                Res.Js.AssemblyWide()
                Res.TzJs
            ]
            ...
        ]
```