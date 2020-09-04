---
title: "WebSharper UI.Next Version 0.1.31.32"
categories: "ui.next,f#,websharper"
abstract: "Websharper UI.Next Version 0.1.31.32 has been released, with bugfixes, and new event & animation functionality."
identity: "4008,77371"
---
WebSharper UI.Next Version 0.1.31.32 has been released on NuGet for your experimentation.

This version begins our push toward better handling of input. In particular, the latest version provides views of mouse and keyboard inputs, and combinators to allow snapshots of views and predicated view updates. 

## View Combinators

```fsharp
    static member SnapshotOn : 'B -> View<'A> -> View<'B> -> View<'B>
    static member UpdateWhile : 'A -> View<bool> -> View<'A> -> View<'A>
```

The new `SnapshotOn` combinator allows snapshots to be taken of a view whenever a 'trigger' view updates. This makes it useful for reacting to events, and has helped with our ongoing work to implement Piglets using UI.Next. 

The `UpdateWhile` combinator provides a view which only updates whenever the value of a given predicate view is true.

While we don't provide first-class discrete event streams just yet, such combinators make it possible to emulate them.

You can find more information in the [Documentation](https://github.com/dotnet-websharper/ui/blob/master/docs/UINext-View.md).

## Input Views

The [Input](https://github.com/dotnet-websharper/ui/blob/master/docs/UINext-Input.md) module provides views of the mouse and keyboard, such as keys and buttons pressed, and the mouse position. This will be expanded in the near future.


## And the rest!

Additionally, there's now support for delayed animation, which is useful when developing visualisations with staggered transitions, and some bugfixes.

You can find samples of the [Mouse](https://websharper-samples.github.io/ui/#/samples/MouseInfo) and [Keyboard](https://websharper-samples.github.io/ui/#/samples/KeyboardInfo) views, as well as the snapshotting and predicated update functions on the [UI.Next samples website](https://websharper-samples.github.io/ui/#/home). As ever, if you've been using UI.Next and have any comments, we'd love to hear them.


## Bugfixes

 *  Animation.Concat sometimes caused "undefined is not a function" errors within JavaScript 
 *  Computation Expression syntax for Views did not compile with WebSharper 
