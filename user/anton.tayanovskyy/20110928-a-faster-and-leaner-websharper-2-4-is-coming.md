---
title: "A Faster and Leaner WebSharper 2.4 is Coming"
categories: "f#,websharper"
abstract: "We are currently finalizing the 2.4 release of WebSharper. This release is mainly concerned with bug fixes and quality-of-implementation improvements. The exciting part is that we are witnessing about 30%-35% generated code size reduction, and a 40% reduction in compilation time. The optimizer is also more capable, featuring local tail call optimization."
identity: "2066,74898"
---
We are currently finalizing the 2.4 release of WebSharper. This release is mainly concerned with bug fixes and quality-of-implementation improvements. The exciting part is that we are witnessing about 30%-35% generated code size reduction, and a 40% reduction in compilation time.

Getting there was rather tedious. One key refactoring was replacing `System.Reflection` with `Mono.Cecil` as the main workhorse for processing F# assemblies. To use Mono.Cecil, and also to avoid some of the nasty reflection bugs in the F# itself, I had to rewrite quotation metadata parsing by having a peek at F# compiler sources. I also had to refactor most of the data structures to stop assuming `System.Reflection` availability.

For getting better code generation, I revised our Core language for simplicity. Both Core and associated optimizer are made simpler. Compiler optimizations are more focused, targeting the common use-cases.

In particular, local tail-call elimination combined with uncurrying finally allows for automated unrolling of recursive functional code into JavaScript loops. The following code, for example, is now safe for stack use:

```fsharp
[<JavaScript>]
let Factorial (n: int) =
    let rec fac n acc =
        match n with
        | 0 | 1 -> acc
        | n -> fac (n - 1) (acc * n)
    fac n 1
```

I am thankful for my colleagues and their effort at building the F# community site, [FPish](https://fpish.net). This codebase provided an invaluable resource for testing and optimizing the compiler on a large project.

I feel that I have learned a lot on this project. It turns out that reading about compilers is not quite the same as writing compilers! You only appreciate this truth after trying. I hardly discovered anything you might call a research contribution, but there are some practical little techniques specialized to F#/.NET that made my life easier and might help you too. Stay tuned for discussions of an approach to maintain and traverse large F# union types, considerations for designing data structures, the memoizing fixpoint pattern, large F# codebase organization ideas, and other things along those lines.
