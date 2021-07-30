---
title: "WebSharper WIG tutorial - how to make JS library extension for WebSharper"
categories: "f#,wig,websharper"
abstract: ""
---

# WebSharper WIG tutorial - how to make JS library extensions for WebSharper

Hello!<br>
In this blog I'm going to show you how to make an extension for WebSharper with WIG (WebSharper Interface Generator). WIG is a powerful tool in WebSharper and with that you can bind JavaScript libraries and access them from F# code. Let's get started!

For this tutorial I will be binding [Day.js](https://day.js.org/en/)

## First steps

Let's create the template project first. You can use the following CLI command:

```bash
dotnet new websharper-ext -o YOUR-EXTENSION-NAME
```

In this case for example:

```bash
dotnet new websharper-ext -o WebSharper.Dayjs
```

It created the WebSharper.Dayjs folder with the following files:

* `Main.fs`
* `WebSharper.Dayjs.fsproj`
* `wsconfig.json`

We mainly work in the `Main.fs`

## Getting to know WIG

When we open the `Main.fs` we'll see this:

```fs
namespace WebSharper.Dayjs

open WebSharper
open WebSharper.JavaScript
open WebSharper.InterfaceGenerator

module Definition =

    let I1 =
        Interface "I1"
        |+> [
                "test1" => T<string> ^-> T<string>
            ]

    let I2 =
        Generic -- fun t1 t2 ->
            Interface "I2"
            |+> [
                    Generic - fun m1 -> "foo" => m1 * t1 ^-> t2
                ]

    let C1 =
        Class "C1"
        |+> Instance [
                "foo" =@ T<int>
            ]
        |+> Static [
                Constructor (T<unit> + T<int>)
                "mem"   => (T<unit> + T<int> ^-> T<unit>)
                "test2" => (TSelf -* T<int> ^-> T<unit>) * T<string> ^-> T<string>
                "radius2" =? T<float>
                |> WithSourceName "R2"
            ]

    let Assembly =
        Assembly [
            Namespace "WebSharper.Dayjs" [
                 I1
                 I2
                 C1
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

This template shows some of the WIG syntax and stuff like `=>`, `=?`. You can find what they mean in the [WebSharper WIG documentation](https://developers.websharper.com/docs/v4.x/fs/wig). For now I will just delete the unneeded parts:

```fs
namespace WebSharper.Dayjs

open WebSharper
open WebSharper.JavaScript
open WebSharper.InterfaceGenerator

module Definition =

    let Assembly =
        Assembly [
            Namespace "WebSharper.Dayjs" [

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

The `Assembly` part is very important. In there goes the classes that will be bound and resources that we need. You can give a `namespace` to each resource which in this case it's `WebSharper.Dayjs`. Because of his I give a different name to the namespace on top of the file, usually `Something.Extension` so here:

```fsharp
namespace WebSharper.Dayjs.Extension

open WebSharper
open WebSharper.JavaScript
open WebSharper.InterfaceGenerator

module Definition =
    ...
```

Let's do some binding now!

## Binding `Day.js`

First things first let's look for the `cdnjs` for Day.js. It can be found [here](https://cdnjs.com/libraries/dayjs). Let's copy the newest version's minified url. We want to put it in the assemblies like this:

```fsharp
let Assembly =
    Assembly [
        Namespace "WebSharper.Dayjs.Resources" [
            yield 
                Resource "DayjsCDN" "https://cdnjs.cloudflare.com/ajax/libs/dayjs/1.10.6/dayjs.min.js"
        ]
        Namespace "WebSharper.Dayjs" [

        ]
    ]
```

Great! This will include a script tag with this source in the project you want to use `WebSharper.Dayjs`. <br> Now let's take a look at the [Day.js documentation](https://day.js.org/docs/en/installation/installation)! After a quick look let's jump to the <b>Parse</b> tab ans start with binding.

First is `Now` subtab which seems to be about the constructor. It accepts a native `Date` object, `undefined` and without parameters works too. Here's how we can bind it:

```fsharp
let Dayjs =
    Class "dayjs"
    |> WithSourceName "DayJs"
    |+> Static [
        Constructor T<unit>
        Constructor T<Date>
        Constructor JS.Undefined
    ]
```

Little explanation to this: the classname should be what you want to see in the JavaScript output. We can see [here](https://day.js.org/docs/en/parse/now) that in JavaScript we make a `Dayjs` object like this:

```js
var now = dayjs()
```
But with `WithSourceName` we can use it in the F# code like this:

```fsharp
let now = DayJs()
```