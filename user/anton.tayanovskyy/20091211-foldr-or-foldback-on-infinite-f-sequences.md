---
title: "Foldr or FoldBack on Infinite F# Sequences"
categories: "f#,haskell,fp,algorithms"
abstract: "The semantics of `foldr` is very simple to remember: it replaces the native `cons` and `nil` of a list with arbitrary computations."
identity: "1453,74663"
---
A noticeable omission in F# standard library is `Seq.foldBack`, or the famous Haskell `foldr`. The semantics of `foldr` is very simple to remember: it replaces the native `cons` and `nil` of a list with arbitrary computations:

```haskell
foldr cons nil []     = nil
foldr cons nil (x:xs) = cons x (foldr cons nil xs)
```

In particular, replacing the native `cons` and `nil` with themselves is always equivalent to the original list, e.g. `forall x: foldr (:) [] x == x`

Surprisingly, the above equation holds for infinite lists as well. This is something important to remember when porting these ideas to F#.

A naive F# translation would use this type:

```fsharp
foldBack : ('T1 -> 'T2 -> 'T2) -> 'T1 -> seq<'T1> -> 'T2
```

However, by being strict in the second argument, cons will now prematurely force the evaluation of infinite sequences.

Here is a more faithful translation using `LazyList` from `FSharp.PowerPack.dll`:

```fsharp
/// Implements the lazy right-to-left fold.
let foldBack (f: 'T1 -> Lazy<'T2> -> 'T2) (z: 'T2) (xs: seq<'T1>) : 'T2 =
    let rec foldr = function
        | LazyList.Nil         -> z
        | LazyList.Cons(x, xs) -> f x (lazy (foldr xs))
    foldr (LazyList.ofSeq xs)
```

Now let us test the code to make sure we have been faithful to Haskell in our translation:

```fsharp
    Seq.FoldBack 
        (fun x xs -> LazyList.consDelayed x (fun () -> Lazy.force xs)) 
        (LazyList.empty ())
        (Seq.initInfinite (fun x -> x))
    |> LazyList.toSeq
    |> Seq.take 10
    |> Seq.toArray
    |> printfn "%A"
```
