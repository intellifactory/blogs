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
        Resource "TimezoneJs" "https://cdnjs.cloudflare.com/ajax/libs/moment-timezone/0.5.33/moment-timezone-with-data.min.js"
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

Great! Let's move on to binding the `moment` object! Now I'm only going to cover the [Parse](https://momentjs.com/docs/#/parsing/) tab in the Moment.js documentation in greater detail, there should be enough examples in it to get you ready for the rest.

--------------

This part is mainly about how to construct a `moment` object. For making it easier to follow, I'm going to use the same structure as the documentation.

### Now

There we can see 4 different constructors:

* unit
* undefined
* array (empty)
* object (empty)

As `undefined` works the same as the `unit` constructor we don't worry about that and leave it out. Let's create a class for the moment object:

```fsharp
let Moment =
    Class "moment"
    |+> Static [

    ]
    |+> Instance [
        
    ]
```

This is how a template class looks like. One thing to note is that what `Class "moment"` does. What you give to `Class` will be what you want to see in the JavaScript output. In the F# code it is a tiny bit different, because WIG will make the first letter capital and when there are characters in the name like `.` or `+` it will switch them to `_`. Well you might ask "Can't I explicitly give a name to use in F#?". WIG provides a method for that, it is done with `WithSourceName "NewName"`. Later on we will use this method but for now it is alright like this.

Let's make the constructors for it:

```fsharp
let Moment =
    Class "moment"
    |+> Static [
        Constructor T<unit>
        Constructor T<int[]>?d
        Constructor T<obj>
    ]
    |+> Instance [
        
    ]
```

As you can see for the `array` I used `int array` because I looked into what type it can be which is the same type of array you use with the native `Date(array)`. Also you might notice the `?d` part. It is makes the array a `named parameter` with the name `d`. We could also add comment to the method that is another helper functionality like the `?d`. We use `WithComment` for that:

```fsharp
Constructor T<unit>
|> WithComment "Initialize with the current time."
Constructor T<int[]>?d
|> WithComment "Create a moment with an array of numbers that mirror the parameters passed to new Date()."
Constructor T<obj>
|> WithComment "Create a moment by specifying some of the units in an object."
```

At last don't forget to add `Moment` or any of the other `CodeModel.Class` you define to the assemblies:

```fsharp
let Assembly =
    Assembly [
        Namespace "WebSharper.Moment.Resources" [
            Res.Js.AssemblyWide()
            Res.TzJs
        ]
        Namespace "WebSharper.Moment" [
            Moment
        ]
    ]
```

### String

Same as before, we add a constructor with a string parameter:

```fsharp
Constructor (T<string>?d)
|> WithComment "Check if the string matches known ISO 8601 formats, then fall back to new Date(string) if a known format is not found, or checking if the string matches with the JSON date."
```

### String + Format

There are quite a few constructors here. Now good thing is we don't have to define all of them if we make use of `optional parameters`. With the `!?` notation we do this:

```fsharp
Constructor (T<string>?d * T<string>?format * !?T<bool>?strict)
|> WithComment "Parse with exact format (with an optional strict parameter)."
Constructor (T<string>?d * T<string>?format * T<string>?language * !?T<bool>?strict)
|> WithComment "Parse with exact format (with optional locale and strict parameters)."
Constructor (T<string>?d * T<string>?format * T<string[]>?languages)
|> WithComment "Parse with exact format (with optional locales)."
```

Defining multiple parameters is done with `*` as you can see.

### String + Formats

Nothing out of the ordinary but note that the 3rd and the 4th parameters are both optional:

```fsharp
Constructor (T<string>?d * T<string[]>?formats * !?T<bool>?strict)
|> WithComment "Parse with multiple format choices (with an optional strict parameter)."
Constructor (T<string>?d * T<string[]>?formats * T<string>?language * !?T<bool>?strict)
|> WithComment "Parse with multiple format choices (with optional locale and strict parameter)."
```

### Special Formats

This is about using predefined formats. Their usage in JavaScript is as follows:

```js
var a = moment.ISO_8601
var b = moment.HTML5_FMT.DATETIME_LOCAL
var c = moment.HTML5_FMT.DATETIME_LOCAL_SECONDS
...
```

The solution for binding this is
* make a read-only property for `ISO_8601` in the Moment class
* make a new class for `moment.HTML5_FMT` that has the properties mentioned in the [documentation](https://momentjs.com/docs/#/parsing/special-formats/)

We can define `read-only` properties with `=?`:

```fsharp
let HTML5ConstantFormats =
    Class "moment.HTML5_FMT"
    |> WithSourceName "HTML5ConstantFormats"
    |+> Static [
        "DATETIME_LOCAL" =? T<string>
        "DATETIME_LOCAL_SECONDS" =? T<string>
        "DATETIME_LOCAL_MS" =? T<string>
        "DATE" =? T<string>
        "TIME" =? T<string>
        "TIME_SECONDS" =? T<string>
        "TIME_MS" =? T<string>
        "WEEK" =? T<string>
        "MONTH" =? T<string>
    ]

let Moment =
    Class "moment"
    |+> Static [
        "ISO_8601" =? T<string>
        ...
    ]
    ...
```

Notice I used `WithSourceName` to give a better name for accessing it from F#.

### Object

Constructor was defined [here](#now) but let's talk about how to use it from F#. You can create objects with `New []` like this:

```fsharp
let myDate : obj = New [
    "hour" => 15
    "minute" => 10
]

let m = Moment(myDate)
```

### Unix Timestamp

Methods can be defined with `=>` followed at some point by `^->`. Let's create the `unix` method:

```fsharp
let Moment =
    Class "moment"
    |+> Static [
        "unix" => T<int> ^-> TSelf
        |> WithComment "Create a moment from a Unix timestamp (seconds since the Unix Epoch)."
    ]
```

### Date

`moment` also accepts the native `Date` as parameter:

```fsharp
Constructor (T<Date>?d)
|> WithComment "Create a Moment with a pre-existing native Javascript Date object."
```

Note that it only works because we have `WebSharper.JavaScript` opened and in it there are bindings for native JavaScript objects.

### Array

Also defined in [Now](#now).

### ASP .NET JSON Date

Same as in [String](#string) (it just provides a different format by default).

### Moment Clone

A constructor with `moment` parameter and an instance method that does the same:

```fsharp
let Moment =
    Class "moment"
    |+> Static [
        Constructor (TSelf?d)
        |> WithComment "Copy constructor."
        ...
    ]
    |+> Instance [
        "clone" => T<unit> ^-> TSelf
        |> WithComment "Returns the clone of the Moment object."
    ]
```

### UTC

Defining the `utc` method goes like this:

```fsharp
"utc" => T<unit> ^-> TSelf
|> WithComment "Current UTC time."
"utc" => T<int>?d ^-> TSelf
|> WithComment "Create a moment by passing an the number of milliseconds since the Unix Epoch (Jan 1 1970 12AM UTC) in UTC."
"utc" => T<int[]>?d ^-> TSelf
|> WithComment "Create a moment with an array of numbers that mirror the parameters passed to new Date() in UTC."
"utc" => T<string>?d ^-> TSelf
|> WithComment "Check if the string matches known ISO 8601 formats, then fall back to new Date(string) if a known format is not found."
"utc" => T<string>?d * T<string>?format * !?T<bool>?strict ^-> TSelf
|> WithComment "Create an UTC moment."
"utc" => T<string>?d * T<string[]>?formats ^-> TSelf
|> WithComment "Create an UTC moment."
"utc" => T<string>?d * T<string>?format * T<string>?language * !?T<bool>?strict ^-> TSelf
|> WithComment "Create an UTC moment."
"utc" => T<string>?d * T<string>?format * T<string[]>?languages ^-> TSelf
|> WithComment "Create an UTC moment."
"utc" => TSelf?d ^-> TSelf
|> WithComment "Create an UTC moment."
"utc" => T<Date>?d ^-> TSelf
|> WithComment "Create an UTC moment."
```

### parseZone

Static method, yet again with optional parameters:

```fsharp
"parseZone" => !?T<string>?d ^-> TSelf
|> WithComment "Parses the time and then sets the zone according to the input string."
"parseZone" => T<string>?d * T<string>?format * !?T<bool>?strict ^-> TSelf
|> WithComment "Parses the time and format (with an optional strict parameter) and then sets the zone according to the input string."
"parseZone" => T<string>?d * T<string[]>?formats ^-> TSelf
|> WithComment "Parses the time and formats and then sets the zone according to the input string."
"parseZone" => T<string>?d * T<string>?format * T<string>?language * !?T<bool>?strict ^-> TSelf
|> WithComment "Parses the time, format and locale (with an optional strict parameter) and then sets the zone according to the input string."
```

### Validation

`moment().isValid()` checks if the moment instance is a valid date. It is a simple instance method:

```fsharp
"isValid" => T<unit> ^-> T<bool>
|> WithComment "Moment applies stricter initialization rules than the Date constructor."
```

This part also talks about `parsingFlags` method in detail so let's do that too! For this I will define the type of object it returns:

```fsharp
let ParsingFlags =
    Class "moment.parsingFlags"
    |> WithSourceName "ParsingFlags"
    |+> Instance [
        "overflow" =? T<int>
        "invalidMonth" =? T<string>
        "empty" =? T<bool>
        "nullInput" =? T<bool>
        "invalidFormat" =? T<bool>
        "userInvalidated" =? T<bool>
        "meridiem" =? T<string>
        "parsedDateParts" =? T<obj[]>
        "unusedTokens" =? T<string[]>
        "unusedInput" =? T<string[]>
    ]

    ...

let Moment =
    Class "moment"
    |+> Static [
        ...
    ]
    |+> Instance [
        "parsingFlags" => T<unit> ^-> ParsingFlags
    ]
```

### Creation Data

Instance method, returns an object:

```fsharp
"creationData" => T<unit> ^-> T<obj>
```

### Defaults

The only new thing here is the `int * string` constructor, the rest are internal things which we don't have to worry about:

```fsharp
Constructor (T<int>?d * T<string>?unit)
|> WithComment "Create a moment by specifying the unit."
```

## Testing

I don't think I have to say that it is recommended to do tests while binding to see if the extension is working as intended. For example we can create a WebSharper Single Page Application project for the tests:

```
dotnet new websharper-spa -lang f# -o WebSharper.Moment.Tests
```

Let's go to the project file it created and just before the `</Project>` end tag, add this:

```xml
<ItemGroup>
  <Reference Include="WebSharper.Moment">
    <HintPath>..\WebSharper.Moment\bin\Debug\netstandard2.0\WebSharper.Moment.dll</HintPath>
  </Reference>
</ItemGroup>
```

This will include the dll of the binding when we do `open WebSharper.Moment` in the tests. You can do your own tests in this but I'm going to show you an example too. Let's modify `wwwroot/index.html` and `Client.fs` a bit:

`wwwroot/index.html`:

```html
<body>
    <h1>Moment js testing</h1>
    <div id="main" ws-children-template="Main">
        <div ws-replace="SomeTest"></div>
    </div>
    <script type="text/javascript" src="Content/WebSharper.Moment.Tests.min.js"></script>
</body>
```

`Client.fs`:

```fsharp
namespace WebSharper.Moment.Tests

open WebSharper
open WebSharper.JavaScript
open WebSharper.JQuery
open WebSharper.UI
open WebSharper.UI.Html
open WebSharper.UI.Client
open WebSharper.UI.Templating
open WebSharper.Moment

[<JavaScript>]
module Client =

    type IndexTemplate = Template<"wwwroot/index.html", ClientLoad.FromDocument>

    [<SPAEntryPoint>]
    let Main () =
        let m = Moment().Format()

        IndexTemplate.Main()
            .SomeTest(
                p [] [text m]
            )
            .Doc()
        |> Doc.RunById "main"

```

Opening `wwwroot/index.html` will show you the results.

## Final words

Now that we finished binding the parsing part the rest shouldn't be much harder. You can try finishing it on your own and see if this little example binding helped. The project can be found on [GitHub](https://github.com/dotnet-websharper/moment) if you want to take a look at it. Thank you for following along and have a nice day!

Tamas Banka [@btamas2000](https://github.com/btamas2000)