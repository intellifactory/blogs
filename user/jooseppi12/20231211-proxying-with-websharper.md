---
title: "Proxying with WebSharper"
categories: "f#,websharper,proxy,fsadvent"
abstract: "Using the Validus library in WebSharper projects"
identity: "-1,-1"
---

I was using the [Validus][1] library in a non-web project, when I realized it would be quite useful to leverage its functionality in the browser as well. I'm using WebSharper and I was not aware of a Validus binding for it, so at first this seemed like a no-go. But here is where WebSharper proxies come to the rescue! But before then, let's start with an alternate approach that feels like the right approach, and then conclude why it's not optimal and why proxies are the right tool after all.

## First attempt: using the source and translating it to JavaScript

As Validus is open source, it's tempting to just compile it to JavaScript. I can grab the source code (say, via a git submodule), create a new WebSharper library project (be sure to install `WebSharper.Templates` via `dotnet new install WebSharper.Templates`, then `dotnet new websharper-lib -lang F# -n AsJavascriptLib`), and add the existing source files to it.

So my project file has a section like this:

```xml
<ItemGroup>
  <Compile Include="../Validus/src/Validus/Core.fs" />
  <Compile Include="../Validus/src/Validus/Validators.fs" />
  <Compile Include="../Validus/src/Validus/Validators.Default.fs" />
  <Compile Include="../Validus/src/Validus/Check.fs" />
  <Compile Include="../Validus/src/Validus/Operators.fs" />
  <None Include="wsconfig.json" />
</ItemGroup>
```

Here is where things start to go south: when I try to compile this, WebSharper tells me that it can't translate Regex.IsMatch from the BCL:

![Error without the proxy](https://i.imgur.com/Z4T81g7.png)


### What is the problem with this?

I can of course rewrite the missing BCL calls and get my JavaScript library, plug it into my JavaScript/TypeScript projects, and all would be OK. But my intention was to use Validus in my WebSharper applications, and by making it into a new WebSharper library I essentially recreated everything as new .NET types, instead of telling the compiler how to translate the existing types (from the Validus assembly) to JavaScript. This means that if we want to reuse functionality both on the client and server side, we are not working with the same types, making things more complicated than what they should be.


## Solution: use a proxy!

A WebSharper proxy is a type that tells the compiler how to compile another .NET type into JavaScript.

Next to library projects, `WebSharper.Templates` also features a `websharper-prx` project template for proxy projects. All you need to proxy .NET libraries is to create your proxy project with `dotnet new websharper-prx -lang F# -n ProxyProject` and add your proxies to it.

Given our earlier prototype, instead of creating a new project, we can simply convert it to a proxy project by changing the content of `wsconfig.json` to this:

```json
{
  "$schema": "https://websharper.com/wsconfig.schema.json",
  "project": "proxy",
  "proxyTargetName": "Validus"
}
```

and adding a `Proxy.fs` to the project file:

```xml
<ItemGroup>
  ...
  <Compile Include="Proxy.fs" />
<ItemGroup>
```

The key here is the `proxyTargetName` setting, which tells the compiler what assembly we want to translate to JavaScript.


## What if I don't have access to a library's source code?

Going back to our Validus experiment, we found that is uses `IsMatch` from `System.Text.RegularExpressions.Regex`, which WebSharper fails to translate. But now, we can tell the compiler how to translate it with the help of the `Proxy` attribute.

`ProxyAttribute` tells the compiler how to translate a piece of .NET code to JavaScript. So to proxy the missing function from `RegularExpressions` that Validus uses, we can simply add the following to `Proxy.fs`, redirecting to the standard JavaScript regex functionality (we will not be trying to bridge the differences between JavaScript and .NET regexes in this article):

```fsharp
open WebSharper.JavaScript

[<Proxy(typeof<System.Text.RegularExpressions.Regex>)>]
type internal RegexProxy =
    static member IsMatch(toMatch: string, pattern: string) = RegExp(pattern).Test toMatch
```

With this tiny addition, we can now build our proxy project and use it along Validus itself in any WebSharper project. In general, you should mark proxies to be `internal` or `private`, as they themselves shouldn't be used/referenced directly. To avoid possible pitfalls, if a proxy is marked as `public`, the compiler will warn about it.


## Let's see it in action!

Alas - we can finally reap the benefits of Validus in a WebSharper app! To use our proxy project we need to include both the original `Validus` library and our new proxy project, like this:

```
<ItemGroup>
  <PackageReference Include="WebSharper" Version="7.0.3.364-beta3" />    
  <PackageReference Include="WebSharper.FSharp" Version="7.0.3.364-beta3" />
  <PackageReference Include="Validus" Version="4.1.3" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="../ProxyProject/ProxyProject.fsproj"></ProjectReference>
</ItemGroup>
```

After that we can take the example from the Validus README:

```fsharp
let ofDto (dto : PersonDto) =
    // A basic validator
    let nameValidator =
        Check.String.betweenLen 3 64

    // Composing multiple validators to form complex validation rules,
    // overriding default error message (Note: "Check.WithMessage.String" as
    // opposed to "Check.String")
    let emailValidator =
        let emailPatternValidator =
            let msg = sprintf "Please provide a valid %s"
            Check.WithMessage.String.pattern @"[^@]+@[^\.]+\..+" msg

        ValidatorGroup(Check.String.betweenLen 8 512)
            .And(emailPatternValidator)
            .Build()

    // Defining a validator for an option value
    let ageValidator =
        Check.optional (Check.Int.between 1 100)

    // Defining a validator for an option value that is required
    let dateValidator =
        Check.required (Check.DateTime.greaterThan DateTime.Now)

    validate {
        let! first = nameValidator "First name" dto.FirstName
        and! last = nameValidator "Last name" dto.LastName
        and! email = emailValidator "Email address" dto.Email
        and! age = ageValidator "Age" dto.Age
        and! startDate = dateValidator "Start Date" dto.StartDate

        // Construct Person if all validators return Success
        return {
            Name = { First = first; Last = last }
            Email = email
            Age = age
            StartDate = startDate }
        }
```

and use it in our `Main` function to populate some content in our HTML:

```fsharp
[<SPAEntryPoint>]
let Main () =
    let scenario1, single_err =
        {
            FirstName = "John"
            LastName  = "John"
            Email     = "john@john"
            Age       = Some 15
            StartDate = Some (DateTime(2035,12,01))
        }
        |> fun d -> d, Person.ofDto d

    let scenario2, multi_err =
        {
            FirstName = "John"
            LastName  = "Doe"
            Email     = "john"
            Age       = None
            StartDate = Some (DateTime(1999,12,01))
        }
        |> fun d -> d, Person.ofDto d

    let scenario3, ok =
        {
            FirstName = "John"
            LastName  = "Doe"
            Email     = "john@doe.com"
            Age       = Some 15
            StartDate = Some (DateTime(2035,12,01))
        }
        |> fun d -> d, Person.ofDto d

    let resolveValidation (r: Result<Person, ValidationErrors>) =
        match r with
        | Result.Ok _ ->
            "OK"
        | Result.Error errors ->
            ValidationErrors.toList errors
            |> String.concat "\n"

    let s1, r1 = JS.Document.QuerySelector "#sample_singleerr > div:first-of-type", JS.Document.QuerySelector "#sample_singleerr > div:last-of-type"
    let s2, r2 = JS.Document.QuerySelector "#sample_multierr > div:first-of-type", JS.Document.QuerySelector "#sample_multierr > div:last-of-type"
    let s3, r3 = JS.Document.QuerySelector "#sample_ok > div:first-of-type", JS.Document.QuerySelector "#sample_ok > div:last-of-type"

    s1.TextContent <- sprintf "%A" <| scenario1.PrettyPrint()
    s2.TextContent <- sprintf "%A" <| scenario2.PrettyPrint()
    s3.TextContent <- sprintf "%A" <| scenario3.PrettyPrint()

    r1.TextContent <- resolveValidation single_err
    r2.TextContent <- resolveValidation multi_err
    r3.TextContent <- resolveValidation ok
```

Combining this with some simple markup (part of `index.html`):

```html
<h2>Single error example</h2>
<div id="sample_singleerr" class="scenario">
    <div></div>
    <div class="error"></div>
</div>
<h2>Multi error example</h2>
<div id="sample_multierr" class="scenario">
    <div></div>
    <div class="error"></div>
</div>
<h2>OK example</h2>
<div id="sample_ok" class="scenario">
    <div></div>
    <div class="ok"></div>
</div>
```

We will get the validation by `Validus` running in our browser with WebSharper:

![Image of sample site](https://i.imgur.com/H2gYdbG.png)


## Closure

All the code used in this blog entry can be found in [this repository][2], and the sample project is also deployed live [here][3]. Please note that in order to run the application on your machine, you will need to add the WebSharper GitHub packages feed to your NuGet sources - see [this page][5] for more details.

Thanks to Sergey Tihon for organizing [F# Advent][4] in 2023 again, and I hope that you find this article useful!

[1]: https://github.com/pimbrouwers/Validus
[2]: https://github.com/jooseppi12/fsadvent2023
[3]: https://jooseppi12.github.io/fsadvent2023
[4]: https://sergeytihon.com/2023/10/28/f-advent-calendar-in-english-2023/
[5]: https://docs.websharper.com/basics/nuget/
