---
title: "Proxying with WebSharper"
categories: "f#,websharper,proxy,fsadvent"
abstract: ""
identity: "-1,-1"
---

I was using the [Validus][1] library in a non-web project, when I realized it would be quite useful to leverage the functionality of it in the browser as well, but as I'm using WebSharper and the library is not built with it, at first it seems like a no-go. But this is the reason why WebSharper's compiler is extendable with proxies! But before utilizing them, let's start at the beggining with an approach, that feels like it should work, but in the end it's not optimal.

## Using the source and translating it to javascript

As Validus is open source, I can just grab the source code (via git submodules), create a new project from WebSharper.Templates via `dotnet new websharper-lib -lang F# -n AsJavascriptLib` and reference the source files and compile with WebSharper. At first this sounds like something that should work and in simple cases would be enough.

So our project file would contain a section like this:

```
<ItemGroup>
  <Compile Include="../Validus/src/Validus/Core.fs" />
  <Compile Include="../Validus/src/Validus/Validators.fs" />
  <Compile Include="../Validus/src/Validus/Validators.Default.fs" />
  <Compile Include="../Validus/src/Validus/Check.fs" />
  <Compile Include="../Validus/src/Validus/Operators.fs" />
  <Compile Include="Proxy.fs" />
  <None Include="wsconfig.json" />
</ItemGroup>
```

We are referencing the Validus source code through the submodule and create a library with WebSharper. You can see that there is a `Proxy.fs` file also included, let's ignore that for now, I will talk about that in a couple sections later.


### What is the problem with this?

But there is a big problem with this approach: We are essentially creating a new type instead of telling the compiler how to translate the existing types to JavaScript. This means that if we want to reuse functionality both on the client and server side, we are not working with the same types making things quite complicated for us.

## Use a proxy project!

WebSharper.Templates also offers a `websharper-prx` project type that we can create via `dotnet new websharper-lib -lang F# -n ProxyProject`. What it does is instead of generating a new type for us, that would shadow the original type, it creates metadata so that the WebSharper compiler can use the `Validus` code during JS translation. We would do the same steps to include the source code of

Alternatively you can convert the existing library project to a proxy one, with changing the content of `wsconfig.json` to this:

```json
{
  "$schema": "https://websharper.com/wsconfig.schema.json",
  "project": "proxy",
  "proxyTargetName": "Validus"
}
```

The key here is the `proxyTargetName` setting, which tells the compiler what Assembly we want to translate to javascript.

## What if I don't have access to the source code

This is the part where I want to get back to that extra file in the first part of the article. `Validus` is using `System.Text.RegularExpressions.Regex.IsMatch` function, which at the moment is not getting translated by the WebSharper compiler. But we can tell the compiler how to translate this with the help of the `ProxyAttribute`.

The `ProxyAttribute` works quite similarly to the proxy project, it tells the compiler how to translate a unit of code to JavaScript. So to proxy the missing function from RegularExpressions to use Validus, we can just add the following to a file and include it in the project:

```fsharp
[<Proxy(typeof<System.Text.RegularExpressions.Regex>)>]
type internal RegexProxy =
    static member IsMatch(toMatch: string, pattern: string) = RegExp(pattern).Test toMatch
```

With the above in our our proxy project, we can build our proxy project to be used in the browser. It is recommended to mark the proxy implementation to be `internal` or `private` as it does not provide any usable code, it is only operating on the metadata level of the WebSharper compilation. If the proxy type is marked as `public` the compiler will give a warning about this.

## Let's see it in action!

To use our proxy project we need to include both the original `Validus` library and the our new proxy project, like this:

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

After that we can take the example from the README of `Validus`:

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

and use it in our `Main` function to populate some content in our html:

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
        } |> fun d -> d, Person.ofDto d

    let scenario2, multi_err =
        {
            FirstName = "John"
            LastName  = "Doe"
            Email     = "john"
            Age       = None
            StartDate = Some (DateTime(1999,12,01))
        } |> fun d -> d, Person.ofDto d

    let scenario3, ok =
        {
            FirstName = "John"
            LastName  = "Doe"
            Email     = "john@doe.com"
            Age       = Some 15
            StartDate = Some (DateTime(2035,12,01))
        } |> fun d -> d, Person.ofDto d

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

Combining this with our simple html:

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

All the code used in this blog entry can be found at [my repository][2] and the sample project is deployed live at [here][3]

Thanks to Sergey Tihon for organizing the [F# Advent Calendar][4] in 2023 again and I hope I could show something new for the readers!

[1]: https://github.com/pimbrouwers/Validus
[2]: https://github.com/jooseppi12/fsadvent2023
[3]: jooseppi12.github.io/fsadvent2023
[4]: https://sergeytihon.com/2022/10/28/f-advent-calendar-in-english-2022/
