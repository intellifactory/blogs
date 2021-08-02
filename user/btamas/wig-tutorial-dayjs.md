---
title: "Creating a WebSharper binding for Day.js"
categories: "f#,wig,websharper"
abstract: ""
---

# Creating a WebSharper binding for `Day.js`

Hello!
In this blog I'm going to show you how to make an extension for WebSharper with WIG (WebSharper Interface Generator). WIG is a powerful tool in WebSharper and with that you can bind JavaScript libraries and access them from F# code. Let's get started!

For this tutorial I will be binding [Day.js](https://day.js.org/en/)

## First steps

Let's create the template project first. You can use the following CLI command:

```bash
dotnet new websharper-ext -o YOUR-EXTENSION-NAME
```

In this case for example:

```bash
dotnet new websharper-ext -o WebSharper.DayJs
```

It created the WebSharper.DayJs folder with the following files:

* `Main.fs`
* `WebSharper.DayJs.fsproj`
* `wsconfig.json`

We mainly work in `Main.fs`

## Getting to know WIG

When we open the `Main.fs` we'll see this:

```fs
namespace WebSharper.DayJs

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
            Namespace "WebSharper.DayJs" [
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
namespace WebSharper.DayJs

open WebSharper
open WebSharper.JavaScript
open WebSharper.InterfaceGenerator

module Definition =

    let Assembly =
        Assembly [
            Namespace "WebSharper.DayJs" [

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

The `Assembly` part is very important. In there goes the classes that will be bound and resources that we need. You can give a `namespace` to each resource which in this case it's `WebSharper.DayJs`. Because of this I give a different name to the namespace on top of the file, usually `Something.Extension` so here:

```fsharp
namespace WebSharper.DayJs.Extension

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
        Namespace "WebSharper.DayJs.Resources" [
            yield 
                Resource "DayJsCDN" "https://cdnjs.cloudflare.com/ajax/libs/dayjs/1.10.6/dayjs.min.js"
        ]
        Namespace "WebSharper.DayJs" [

        ]
    ]
```

Great! This will include a script tag with this source in the project you want to use `WebSharper.DayJs`. Now let's take a look at the [Day.js documentation](https://day.js.org/docs/en/installation/installation)! After a quick look let's jump to the <b>Parse</b> tab ans start with binding.

First is `Now` subtab which seems to be about the constructor. It accepts a native `Date` object and without parameters works too (It also accepts `undefined` but since it says it will translate to a unit constructor there is no need to put that in). Here's how we can bind it:

```fsharp
let DayJs =
    Class "dayjs"
    |> WithSourceName "DayJs"
    |+> Static [
        Constructor T<unit>
        Constructor T<Date>
    ]
```

Little explanation to this: the classname should be what you want to see in the JavaScript output. We can see [here](https://day.js.org/docs/en/parse/now) that in JavaScript we make a `DayJs` object like this:

```js
var now = dayjs()
```
But with `WithSourceName` we can use it in the F# code like this:

```fsharp
let now = DayJs()
```

Moving on to the `String` subtab we can see it is also a constructor that accpets `ISO 8601` format strings but we just use plain string for it:

```fsharp
let DayJs =
    Class "dayjs"
    |> WithSourceName "DayJs"
    |+> Static [
        Constructor T<unit>
        Constructor T<Date>
        Constructor T<string>
    ]
```

Next, in the `String + Format` we see that this constructor accepts multiple arguments, there are both required and optional parameters too. Note that

* First param: `string`, required
* Second param: either `string` or `string array`, required
* Third param: either `string` or `boolean`, optional
* Fourth param: `boolean` (only if the third param was `string`), optional

With this in mind, the binding goes like this:

```fsharp
let DayJs =
    Class "dayjs"
    |> WithSourceName "DayJs"
    |+> Static [
        Constructor T<unit>
        Constructor T<Date>
        Constructor T<string>
        Constructor (T<string> * (T<string> + !| T<string>) * !? T<bool>)
        Constructor (T<string> * (T<string> + !| T<string>) * T<string> * !? T<bool>)
    ]
```

Quick explanation: `Constructor` only accepts one parameter so when it needs more put them in brackets. `*` is used when there are multiple parameters, `+` is for the "either" case, `!?` means the parameter after that is optional and `!|` is used when the parameter is an array. For information about WIG notations, please visit the [WIG documentation](https://developers.websharper.com/docs/v4.x/fs/wig).

There is one problem though. As you can see, this constructor depends on a plugin called `CustomParseFormat`. In JavaScript it should be used like this:

```js
var customParseFormat = require('dayjs/plugin/customParseFormat')

dayjs.extend(customParseFormat)
dayjs('12-25-1995', 'MM-DD-YYYY')
```

This is where this tutorial gets a little advanced. A possible solution to this uses `RequiresExternal`. It would look like this:

```fsharp
Constructor (T<string> * (T<string> + !| T<string>) * !? T<bool>)
|> RequiresExternal [T<MY_RESOURCE_TYPE>]
```

One thing to note is that the above mentioned `MY_RESOURCE_TYPE` has to implement the `IResource` interface. Let's make a WebSharper Library project for this!

```
dotnet new websharper-lib -lang f# -o WebSharper.DayJs.Helpers
```

After taking a look at [dotnet-websharper/core](https://github.com/dotnet-websharper/core), [this](https://github.com/dotnet-websharper/core/blob/master/src/compiler/WebSharper.Core/Resources.fs) is the file we are looking for since WIG uses the `BaseResource` type in it. We can just copy the necessary code which implements the IResource:

```fsharp
namespace WebSharper.DayJs.Helpers

open WebSharper
open WebSharper.Core.Resources
open System

module DayJsHelpers =

    let cleanLink dHttp (url: string) =
        if dHttp && url.StartsWith("//")
            then "http:" + url
            else url

    let link dHttp (html: HtmlTextWriter) (url: string) =
        if not (String.IsNullOrWhiteSpace(url)) then
            html.AddAttribute("type", Core.ContentTypes.Text.Css.Text)
            html.AddAttribute("rel", "stylesheet")
            html.AddAttribute("href", cleanLink dHttp url)
            html.RenderBeginTag "link"
            html.RenderEndTag()
            html.WriteLine()

    let script dHttp (html: HtmlTextWriter) isModule (url: string) =
        if not (String.IsNullOrWhiteSpace(url)) then
            html.AddAttribute("src", cleanLink dHttp url)
            html.AddAttribute("type", if isModule then Core.ContentTypes.Text.Module.Text else Core.ContentTypes.Text.JavaScript.Text)
            html.AddAttribute("charset", "UTF-8")
            html.RenderBeginTag "script"
            html.RenderEndTag()

    type Kind =
        | Basic of string
        | Complex of string * list<string>

    let tryFindWebResource (t: Type) (spec: string) =
        let ok name = name = spec || (name.StartsWith spec && name.EndsWith spec)
        t.Assembly.GetManifestResourceNames()
        |> Seq.tryFind ok

    let tryGetUriFileName (u: string) =
        if u.StartsWith "http:" || u.StartsWith "https:" || u.StartsWith "//" then
            let parts = u.Split([| '/' |], StringSplitOptions.RemoveEmptyEntries)
            Array.tryLast parts
        else
            None

    type DayJsResource(kind: Kind) as this =
        let self = this.GetType()
        let name = self.FullName

        new (spec: string) =
            new DayJsResource(Basic spec)

        new (b: string, x: string, [<System.ParamArray>] xs: string []) =
            new DayJsResource(Complex(b, x :: List.ofArray xs))

        member this.GetLocalName() =
            name.Replace('+', '.').Split('`').[0]

        interface IResource with
            member this.Render ctx =
                let dHttp = ctx.DefaultToHttp
                let isLocal = ctx.GetSetting "UseDownloadedResources" |> Option.exists (fun s -> s.ToLower() = "true")
                let localFolder isCss f =
                    ctx.WebRoot + 
                    (if isCss then "Content/WebSharper/" else "Scripts/WebSharper/") + this.GetLocalName() + "/" + f
                match kind with
                | Basic spec ->
                    let mt = 
                        if spec.EndsWith ".css" then Css 
                        elif spec.EndsWith ".mjs" then JsModule 
                        else Js
                    let r =
                        match ctx.GetSetting name with
                        | Some url -> RenderLink url
                        | None ->
                            match tryFindWebResource self spec with
                            | Some e -> Rendering.GetWebResourceRendering(ctx, self, e)
                            | None ->
                                if isLocal then
                                    match tryGetUriFileName spec with
                                    | Some f ->
                                        RenderLink (localFolder (mt = Css) f)
                                    | _ ->
                                        RenderLink spec
                                else
                                    RenderLink spec
                    fun writer -> r.Emit(writer, mt, dHttp)
                | Complex (b, xs) ->
                    let b = defaultArg (ctx.GetSetting name) b
                    let urls =
                        xs |> List.map (fun x ->
                            let url = b.TrimEnd('/') + "/" + x.TrimStart('/')
                            url, url.EndsWith ".css"     
                        )  
                    let urls = 
                        if isLocal then 
                            urls |> List.map (fun (u, isCss) ->
                                match tryGetUriFileName u with
                                | Some f ->
                                    localFolder isCss f, isCss
                                | _ ->
                                    u, isCss
                            )
                        else urls
                    fun writer ->
                        for url, isCss in urls do
                            if isCss then
                                link dHttp (writer Styles) url
                            else script dHttp (writer Scripts) false url
```

Basicly what is important in this code is that one of its functionality is making a `script` element in the html's `head`. [Here](https://day.js.org/docs/en/plugin/loading-into-browser) it is stated that we can use the `extend` method like this:

```js
dayjs.extend(window.dayjs_plugin_advancedFormat)
```

What we need is to write a script element with the above line, replacing `advancedFormat` with the appropriate plugin name. This method should do the job:

```fsharp
let extendScript (advFormat: string) (html: HtmlTextWriter) =
    html.RenderBeginTag "script"
    html.Write(sprintf "dayjs.extend(window.dayjs_plugin_%s)" advFormat)
    html.RenderEndTag()
```

Well, how do we get the `advancedFormat`? Looking at the [CDNJS](https://cdnjs.com/libraries/dayjs) links every link has this format: `https://cdnjs.cloudflare.com/ajax/libs/dayjs/1.10.6/dayjs.min.js`. Using `System.String().Split()` we can get the name easily:

```fsharp
let splitCDN =
    url.Split('/')
    |> Array.last
    |> fun x -> x.Split('.')
    |> Array.head
```

Now let's extend the type with this logic:

```fsharp
interface IResource with
    member this.Render ctx =
        let dHttp = ctx.DefaultToHttp
        let isLocal = ctx.GetSetting "UseDownloadedResources" |> Option.exists (fun s -> s.ToLower() = "true")
        let localFolder isCss f =
            ctx.WebRoot + 
            (if isCss then "Content/WebSharper/" else "Scripts/WebSharper/") + this.GetLocalName() + "/" + f
        match kind with
        | Basic spec ->
            let mt = 
                if spec.EndsWith ".css" then Css 
                elif spec.EndsWith ".mjs" then JsModule 
                else Js
            match ctx.GetSetting name with
            | Some url ->
                RenderLink url
                |> fun r ->
                    fun writer ->
                        let splitCDN =
                            url.Split('/')
                            |> Array.last
                            |> fun x -> x.Split('.')
                            |> Array.head
                        r.Emit(writer, mt, dHttp)
                        extendScript splitCDN (writer Scripts)
            | None ->
                match tryFindWebResource self spec with
                | Some e -> Rendering.GetWebResourceRendering(ctx, self, e)
                | None ->
                    if isLocal then
                        match tryGetUriFileName spec with
                        | Some f ->
                            RenderLink (localFolder (mt = Css) f)
                        | _ ->
                            RenderLink spec
                    else
                        RenderLink spec
                |> fun r ->
                    fun writer ->
                        let splitCDN =
                            spec.Split('/')
                            |> Array.last
                            |> fun x -> x.Split('.')
                            |> Array.head
                        r.Emit(writer, mt, dHttp)
                        extendScript splitCDN (writer Scripts)
        | Complex (b, xs) ->
            let b = defaultArg (ctx.GetSetting name) b
            let urls =
                xs |> List.map (fun x ->
                    let url = b.TrimEnd('/') + "/" + x.TrimStart('/')
                    url, url.EndsWith ".css"     
                )  
            let urls = 
                if isLocal then 
                    urls |> List.map (fun (u, isCss) ->
                        match tryGetUriFileName u with
                        | Some f ->
                            localFolder isCss f, isCss
                        | _ ->
                            u, isCss
                    )
                else urls
            fun writer ->
                for url, isCss in urls do
                    if isCss then
                        link dHttp (writer Styles) url
                    else
                        let splitCDN =
                            url.Split('/')
                            |> Array.last
                            |> fun x -> x.Split('.')
                            |> Array.head
                        script dHttp (writer Scripts) false url
                        extendScript splitCDN (writer Scripts)
```

Now to make the resource types we need:

```fsharp
type MainResource() =
    inherit BaseResource("https://cdnjs.cloudflare.com/ajax/libs/dayjs/1.10.6/dayjs.min.js")

[<Require(typeof<MainResource>)>]
type CustomParseFormatResource() =
    inherit DayJsResource("https://cdnjs.cloudflare.com/ajax/libs/dayjs/1.10.6/plugin/customParseFormat.min.js")
```

Jumping back to `WebSharper.DayJs/Main.fs` we have to make the following modifications:

```fsharp
let DayJs =
    Class "dayjs"
    |> RequiresExternal [T<WebSharper.DayJs.Helpers.DayJsHelpers.MainResource>]
    |> WithSourceName "DayJs"
    |+> Static [
        Constructor T<unit>
        Constructor T<Date>
        Constructor T<string>
        Constructor (T<string> * (T<string> + !| T<string>) * !? T<bool>)
        |> RequiresExternal [T<WebSharper.DayJs.Helpers.DayJsHelpers.CustomParseFormatResource>]
        Constructor (T<string> * (T<string> + !| T<string>) * T<string> * !? T<bool>)
        |> RequiresExternal [T<WebSharper.DayJs.Helpers.DayJsHelpers.CustomParseFormatResource>]
        Constructor (T<int>)
    ]

let Assembly =
    Assembly [
        // Namespace "WebSharper.DayJs.Resources" [
        //     // DayJsResources.CustomParseFormatResource
            
        // ]
        Namespace "WebSharper.DayJs" [
            FromStandardObject
            FromStandardPluralObject
            DayJs
        ]
    ]
```

Now for the final part let's modify the project file a bit. Add the following code to the end of the project file (just before the `</Project>`):

```xml
<ItemGroup>
  <Reference Include="WebSharper.DayJs.Helpers">
    <HintPath>..\WebSharper.DayJs.Helpers\bin\Debug\netstandard2.0\WebSharper.DayJs.Helpers.dll</HintPath>
  </Reference>
</ItemGroup>
```

This will include the helpers' dll that is needed when we want to use the resources from it.

Great! Now let's do some testing with it. Let's create a WebSharper SPA project for the tests, use the following CLI command:

```
dotnet new websharper-spa -lang f# -o WebSharper.DayJs.Tests
```

For the record, the directory structure should look something like this:

* WebSharper.DayJs
* WebSharper.DayJs.Helpers
* WebSharper.DayJs.Tests

Add the same kind of changes to the test project file but with `WebSharper.DayJs.dll`:

```xml
<ItemGroup>
  <Reference Include="WebSharper.DayJs">
    <HintPath>..\WebSharper.DayJs\bin\Debug\netstandard2.0\WebSharper.DayJs.dll</HintPath>
  </Reference>
</ItemGroup>
```

Now let's delete the unnecessary parts from the tests' `Client.fs`! It should look like this:

```fsharp
namespace WebSharper.DayJs.Tests

open WebSharper
open WebSharper.JavaScript
open WebSharper.JQuery
open WebSharper.UI
open WebSharper.UI.Client
open WebSharper.UI.Templating

[<JavaScript>]
module Client =

    type IndexTemplate = Template<"wwwroot/index.html", ClientLoad.FromDocument>

    [<SPAEntryPoint>]
    let Main () =

        IndexTemplate.Main()
            .Doc()
        |> Doc.RunById "main"

```

I'll also modify the `index.html` a bit. My preference is that I move the index.html out of the `wwwroot` folder, delete the folder and modify these files accordingly:

* In `Client.fs` change `IndexTemplate`:
    * ```fsharp
      type IndexTemplate = Template<"index.html", ClientLoad.FromDocument>
      ```
* In `wsconfig.json` change `outputDir`:
    * ```json
      "outputDir": "Content"
      ```

Nice! Now let's print some dates as test! First we go to `index.html` and do something like this:

```html
<body>
    <h1>DayJs tests</h1>
    <div id="main" ws-children-template="Main">
        <div ws-replace="DatesAsString"></div>
    </div>
    <script type="text/javascript" src="Content/WebSharper.DayJs.Tests.min.js"></script>
</body>
```

I won't go into too much detail on how to make WebSharper template html files, please visit the [WebSharper documentation](https://developers.websharper.com/docs/v4.x/fs/overview). Now go to the `Client.fs` and make the following changes:

```fsharp
open WebSharper.UI.Html
open WebSharper.DayJs
...

    [<SPAEntryPoint>]
    let Main () =

        let someDate = DayJs("2019-01-25")
        let formattedDate = DayJs("12-25-1995", "asdfasdsadasdasd")

        IndexTemplate.Main()
            .DatesAsString(
                Doc.Concat [
                    p [] [text someDate]
                    p [] [text formattedDate]
                ]
            )
            .Doc()
        |> Doc.RunById "main"

```

The reason for `formattedDate`'s second parameter is that we'd like to test if the plugin is imported as well so this should give us `Invalid date` as a result.

But this will not compile just yet because the type of both `someDate` and `formattedDate` is `DayJs` and `text` requires a `string` as parameter. So let's bind a function that converts DayJs objects to string. There it is, in the [Display/As String tab](https://day.js.org/docs/en/display/as-string). Add the `toString()` method:

```fsharp
let DayJs =
    Class "dayjs"
    |> RequiresExternal [T<WebSharper.DayJs.Helpers.DayJsHelpers.MainResource>]
    |> WithSourceName "DayJs"
    |+> Static [
        Constructor T<unit>
        Constructor T<Date>
        Constructor T<string>
        Constructor (T<string> * (T<string> + !| T<string>) * !? T<bool>)
        |> RequiresExternal [T<WebSharper.DayJs.Helpers.DayJsHelpers.CustomParseFormatResource>]
        Constructor (T<string> * (T<string> + !| T<string>) * T<string> * !? T<bool>)
        |> RequiresExternal [T<WebSharper.DayJs.Helpers.DayJsHelpers.CustomParseFormatResource>]
        Constructor (T<int>)
    ]
    |+> Instance [
        "toString" => T<unit> ^-> T<string>
    ]
```

If we add the `.ToString()` to `someDate` and `formattedDate` in the tests it will now compile and if you open the `index.html` after build you will see one is a correct date while the other is invalid. This confirms that the binding is good so far.

From now on I will only give explanations when we encounter problems where the solution has not yet been seen in this tutorial.

In [Parse/Object](https://day.js.org/docs/en/parse/object) we can see that the constructor can also accept an object that contains optional values like `year`, `month`, etc. When we have to bind something like this, use `Pattern.Config`:

```fsharp
let FromStandardObject =
    Pattern.Config "FromStandardObject" {
        Required = []
        Optional = [
            "years", T<int>
            "months", T<int>
            "date", T<int>
            "hours", T<int>
            "minutes", T<int>
            "seconds", T<int>
            "milliseconds", T<int>
        ]
    }    
```

Add it to the constructor:

```fsharp
Constructor FromStandardObject
|> RequiresExternal [T<WebSharper.DayJs.Helpers.DayJsHelpers.ObjectSupport>]
```
Also do not forget when you make new classes like with `Pattern.Config` to add it to the namespaces:

```fsharp
Namespace "WebSharper.DayJs" [
    FromStandardObject
    DayJs
]
```

