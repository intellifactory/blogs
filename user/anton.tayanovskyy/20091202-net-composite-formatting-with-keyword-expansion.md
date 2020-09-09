---
title: ".NET Composite Formatting with Keyword Expansion"
categories: "f#,dotnet"
abstract: ""
identity: "1458,74668"
---
[Composite Formatting](https://msdn.microsoft.com/en-us/library/txafckwd%28VS.95%29.aspx) is a feature of .NET framework that comes handy even for F# programmers. Yes, `Printf`-style formatting generally is much nicer with F#, but there are situations where the format string is not available statically. It can, for instance, be coming from a configuration file.

One common issue with Composite Formatting is that it is not immediately obvious how to expand named arguments. Fortunately, all the pieces of the puzzle are there.

Just a little bit of F#:

```fsharp
module Format =
    open System
    open System.Collections.Generic

    let private Split (s: string) =
        match s.IndexOf '|' with
        | -1 -> (s, "")
        | n  -> (s.Substring(0, n), s.Substring(n + 1))

    type Table<'T>(dict: IDictionary<string, 'T>) =
        new (pairs: seq<string * 'T>) = 
            new Table<'T>(Map.ofSeq pairs)

        interface IFormattable with
            member this.ToString(format, _) =
                let (key, def) = Split format
                if dict.ContainsKey key then
                    dict.[key].ToString()
                else
                    def
```

Now we can use string keys (and default values) to expand on keywords within the format strings. This is handy:

```fsharp
let fmt = "{0:schema|http}://{0:domain}{0:path}"
System.String.Format(fmt, Format.Table ["domain", "example.com"])
System.String.Format(fmt, Format.Table ["domain", "example.com"; "path", "/products"])
System.String.Format(fmt, Format.Table ["schema", "https"; "domain", "example.com"])
```

You can also pass `Dictionary` and `Map` objects to the `Format.Table` constructor.

Even better, Composite Formatting is available not only in `String.Format` but also in other places such as `TextWriters`.

As a functional programmer working with F#, I keep discovering basic .NET framework features. Even though this use of Composite Formatting must be trivial, I hope some of you will find it useful.
