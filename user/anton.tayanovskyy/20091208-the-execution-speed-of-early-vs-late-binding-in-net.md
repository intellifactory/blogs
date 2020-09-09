---
title: "The Execution Speed of Early vs Late Binding in .NET"
categories: "f#,optimization,dotnet"
abstract: ""
identity: "1456,74666"
---
This little post documents one of my little experiments with F#, as I am educating myself on the .NET Framework fundamentals.

The interesting issue is the execution speed of late vs early-bound code. Open the F# interactive and try this out.

```fsharp
/// A dummy type.
type T = 
    /// This is the method we want to call.
    static member F x = x + 1
```

Turn on timing in the Interactive:

```fsharp
#time
```

First, let as assume we know the type and the method signature at compile time. This is early, static binding, and the result is fast:

```fsharp
for i = 0 to 1000000 do
    T.F 1 
    |> ignore
```

Now suppose we do not know type `T` at compile time, but rather have a `System.Type` object to represent it.

We could then use reflection to invoke the method, as below, but it is slow, several orders of magnitude slower in fact.

```fsharp
let m = typeof<T>.GetMethod("F")
for i = 0 to 1000000 do
    m.Invoke(null, [| box 1 |])
    |> ignore
```

If we do not know the type T, but do know the method signature, we can do a lot better using delegates. This is fast:

```fsharp
type F = delegate of int -> int
let  f = System.Delegate.CreateDelegate(typeof<F>, typeof<T>, "F") :?> F
for i = 0 to 1000000 do
    f.Invoke 1 
    |> ignore
```
