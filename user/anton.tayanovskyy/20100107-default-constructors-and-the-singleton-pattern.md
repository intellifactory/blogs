---
title: "Default Constructors and the Singleton Pattern"
categories: "f#,dotnet"
abstract: "In this blog I will demonstrate a common F# idiom for passing values through the type system."
identity: "1452,74662"
---
In this blog I will demonstrate a common F# idiom for passing values through the type system.

The need for this usually comes as you are trying to trick F# into doing something advanced. Typically, you write a function F accepting a type paremeter 'T and expecting to use some functionality that 'T provides. Passing instances of 'T is not always a good option, as passing types is much cheaper syntactically (thanks to the type inferencer).

This can be done by constraining 'T to implement an interface I. However, unlike Java, .NET interfaces cannot constrain static members. Thankfully, this can be resolved by default constructor constraints and on the type coupled with type initialization.

```fsharp
    type IFoo =
        abstract member Bar: unit -> unit

    let F<'T when 'T :> IFoo and 'T : (new : unit -> 'T)>() =
        let t = new 'T()
        t.Bar()
```

The remaining problem is the generated GC load, as new instances of 'T are created at every invocation of F.

A way to go here is to use the [Singleton](https://msdn.microsoft.com/en-us/library/ms998558.aspx) pattern:

```fsharp
    [<Sealed>]
    type Singleton<'T when 'T : (new : unit -> 'T)> private () =
        static let instance : 'T = new 'T()
        static member Instance : 'T = instance

    type IFoo =
        abstract member Bar: unit -> unit

    let F<'T when 'T :> IFoo and 'T : (new : unit -> 'T)>() =
        Singleton<'T>.Instance.Bar()
```

The GC time advantages of using the singleton can be easily measured:

```fsharp
module A = 
    let F<'T when 'T :> IFoo and 'T : (new : unit -> 'T)>() =
        (new 'T() :> IFoo).Bar()

    let G<'T when 'T :> IFoo and 'T : (new : unit -> 'T)>() =
        (Singleton<'T>.Instance :> IFoo).Bar()

#time    

for f in 0 .. 10000000 do
    A.F<Foo>()

// Real: 00:00:01.465, CPU: 00:00:01.453, GC gen0: 114, gen1: 0, gen2: 0

for f in 0 .. 10000000 do
    A.G<Foo>()

// Real: 00:00:00.544, CPU: 00:00:00.546, GC gen0: 0, gen1: 0, gen2: 0
```
