---
title: "Generic Workflow Builders (Monads) in F#"
categories: "f#,haskell,fp"
abstract: "This blog post is about a quick and dirty encoding of Haskell type classes in F#."
identity: "1454,74664"
---
This blog post is about a quick and dirty encoding of Haskell type classes in F#. With the ongoing work on the WebSharper project, we are currently very interested in coaxing the .NET type system to support writing code that is generalized over monads and applicative functors. I thank [Sandro Magi](https://higherlogics.blogspot.com/) and refer to his excellent blog posts with C# code that explain the general technique. Here I will just give sample code.

The problem to solve is to define a generic workflow (monad) builder, in such a way that the user can write code parameterized over the monad. The solution sacrifices type safety by using downcasts. However, with very mild assumptions, the downcasts are always safe. The technique scales to other Haskell type classes such as `Applicative`.

```fsharp
type IMonad<'M when 'M :> IMonad<'M> and 'M : (new : unit -> 'M)> =
    abstract member Return<'T> : 'T -> IMonad<'T,'M>
    abstract member Bind<'T1,'T2> : IMonad<'T1,'M> * ('T1 -> IMonad<'T2,'M>) -> IMonad<'T2,'M>    
    
and IMonad<'T,'M when 'M :> IMonad<'M>> = interface end

[<Sealed>]
type MonadBuilder<'M when 'M :> IMonad<'M> 
                     and 'M : (new : unit -> 'M)> private () =
    
    let m = new 'M() :> IMonad<'M>
    static let self = new MonadBuilder<'M>()
    
    static member Instance : MonadBuilder<'M> = self

    member this.Return<'T>(x: 'T) : IMonad<'T,'M> = 
        m.Return x

    member this.Bind(x: IMonad<'T1,'M>, f: 'T1 -> IMonad<'T2,'M>) : IMonad<'T2,'M> =
        m.Bind(x, f)

    member this.Delay<'T,'R when 'R :> IMonad<'T,'M>>(f: unit -> IMonad<'T,'M>) : 'R = 
        m.Bind (m.Return (), f) :?> 'R

type Maybe<'T> =
    | Nothing
    | Just of 'T

    interface IMonad<'T,MaybeInstances>

and MaybeInstances () =
    interface IMonad<MaybeInstances> with
        member this.Return x =
            Just x :> _

        member this.Bind(x, f) =
            match x :?> _ with
            | Nothing -> Nothing :> _
            | Just x  -> f x

let maybe = MonadBuilder<MaybeInstances>.Instance

maybe {
    let! x = Just 1
    let! y = Just 5
    return x + y
}
|> printfn "%A"
```

There are several things to notice. First, monad builders can be constructed with `MonadBuilder.Instance` by passing a complying T. Second, monad-generalized functions such as `Control.Monad.Sequence` from Haskell prelude are expressible with the `IMonad` interface pair.

One serious drawback is the inability to define instances on existing types, such as `seq`. This can be worked around by using isomorphic new types, but those require explicit (un)wrapping.

Nevertheless, the technique seems applicable enough. If things go well, the code can make it into a library.
