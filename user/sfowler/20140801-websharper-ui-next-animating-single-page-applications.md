---
title: "WebSharper UI.Next: Animating Single-Page Applications"
categories: "reactivedom,animation,ui.next,f#,websharper"
abstract: "WebSharper.UI.Next provides a powerful declarative animation system. This mini-blog shows how to integrate this with single-page sites and flowlets."
identity: "3985,77338"
---
In my [previous blog](http://fpish.net/blog/SimonJF/id/3965/2014722-structuring-web-applications-with-websharper-ui-next), I was asked about how easy it would be to tie in custom animations to the flowlet I described. In this post I hope to convince you that the answer to that is, well... Pretty easy!

If you haven't checked out [Anton's blog on declarative animation in UI.Next](http://fpish.net/blog/anton.tayanovskyy/id/3964/2014721-websharper-ui-next-declarative-animation), it provides a good introduction and is a bit more in-depth than this one. It'll also help to check out the blog on [structuring single-page applications](http://fpish.net/blog/SimonJF/id/3965/2014722-structuring-web-applications-with-websharper-ui-next).

A brief introduction for those who haven't read up on UI.Next just yet: UI.Next is a dataflow-backed reactive DOM framework for [WebSharper](http://www.websharper.com). By defining reactive variables (which are a bit like reference cells) and views on these (which allow you to 'peer into' the variable as it changes), it's possible to create DOM fragments which change with the variable.

This makes it really good for constructing single-page applications (SPAs), and we've cooked up some patterns and abstractions to make this feel pretty natural. Of course, with SPAs, some kind of animation between page transitions is quite nice -- and very doable. Of course, you could do this with CSS -- but I hope you'll agree that it's nicer and more powerful in F# :)

## Fade Transitions for a Mini-Site

Earlier, I showed a pattern to create mini-sites consisting of multiple pages. To do this, we create an ADT to show the different types of pages you can have, and a rendering function for each. Once this is done, we get a view of the variable, map the rendering function onto it, then embed it into the document.

```fsharp
 // Here we define the minisite.
    let Main () =
        // We first create a variable to hold our current page.
        let m = Var.Create BobsleighHome

        // Here, we render the current page using a view of the current page variable
        let ctx = {Go = Var.Set m}
        View.FromVar m
        |> View.Map (fun pg ->
            match pg with
            | BobsleighHome -> HomePage ctx
            | BobsleighHistory -> History ctx
            | BobsleighGovernance -> Governance ctx
            | BobsleighTeam -> Team ctx)
        |> Doc.EmbedView // Finally, we can embed it to get a Doc
```

Okay, so what we want is for each page to fade out, and a new one to fade in. You can see this live [here](https://websharper-samples.github.io/ui/#/samples/AnimatedBobsleighSite).

To do this, we'll take a look at a few things in the `Animation` module: if you're interested in looking into this a bit more deeply, then check out the [API Reference](https://github.com/dotnet-websharper/ui/blob/master/docs/UINext-Animation.md).

The first thing to do is create an `Anim`, which specifies the basic properties about the animation:

 *  **Interpolation**: How to calculate the intermediate values given the current time. This is parameterised over the type we want to interpolate -- we'll just use the inbuilt `Double` interpolation.
 *  **Easing**: How to "distort" the interpolation, to give more interesting effects. We'll use the `CubicInOut` easing.
 *  **Time**: The amount of time the animation will last. 


Without further ado...

```fsharp
    let fadeTime = 300.0
    let Fade =
        Anim.Simple Interpolation.Double Easing.CubicInOut fadeTime
```

The next step is to define a `Transition`, which will trigger the animation. We want to fade in when the page gets shown (meaning we'll need an enter transition) and out when the page gets removed (meaning we'll need an exit transition). When this happens, we'll want to play the Fade animation, changing the `opacity` style from 0.0 to 1.0 for a fade in, and 1.0 to 0.0 for a fade out.

You'll notice that `Trans.Enter` and `Trans.Exit` take a function providing a value. This allows the transition to depend on a time-varying element -- in this case, since the transition will be the same each time, we don't need this. It comes in really handy for data-driven animations though, as you can see in the [object constancy sample](https://websharper-samples.github.io/ui/#/samples/ObjectConstancy).

```fsharp
    let FadeTransition =
        Trans.Create Fade
        |> Trans.Enter (fun i -> Fade 0.0 1.0)
        |> Trans.Exit (fun i -> Fade 1.0 0.0)
```

The final step is to create an animation attribute, and attach it to the element we want to animate. This is done as follows, using the `Attr.Animated` and `Attr.AnimatedStyle` functions.

```fsharp
let MakePage var pg =
    Doc.Concat [
        NavBar var
        Div [Attr.AnimatedStyle "opacity" FadeTransition (View.Const 1.0) string] [
            pg
        ]
    ]
```

The function firstly takes the appropriate attribute we want to change as a result of the animation -- in the case of a fade, this will be opacity. We then specify our transition, and a constant view to be passed to the transition. Recall that since this animation doesn't depend on state, it's pretty much irrelevant in this case, but can come in handy at other points. The final argument is a function to change the double we're animating to a string, so that it can be displayed as an attribute.

All that's left now is to pipe the pages to the `MakePage` function, which also adds a navigation bar, and we're done!

```fsharp
let Main () =
    let m = Var.Create BobsleighHome
    let ctx = {Go = Var.Set m}
    View.FromVar m
    |> View.Map (fun pg ->
        match pg with
        | BobsleighHome -> HomePage ctx
        | BobsleighHistory -> History ctx
        | BobsleighGovernance -> Governance ctx
        | BobsleighTeam -> Team ctx
        |> MakePage m)
    |> Doc.EmbedView
```

## Animating Flowlets

It's also very possible to combine animations. Flowlets allow us to easily construct "wizard-like" forms, where each page that has been shown may depend on previous pages. Now, a nice thing to do in such a case is to have a swipe-style animation that's triggered upon clicking the "next" button of the form, for example. You can see this example live [here](https://websharper-samples.github.io/ui/#/samples/AnimatedContactFlow).

The first thing to do is to define a new swipe animation and transition, much as before: the code is pretty much the same, but we'll move 400px to the right, so we'll interpolate between 0.0 and 400.0 this time instead. The transition will also only be triggered when the node is removed, so we'll only need an exit transition this time.

```fsharp
// It's a similar story for swipes -- we're wanting to swipe 400px left.
let swipeTime = 300.0
let Swipe =
    Anim.Simple Interpolation.Double Easing.CubicInOut swipeTime

// We're only swiping out -- so we only specify an exit transition here
let SwipeTransition =
    Trans.Create Swipe
    |> Trans.Exit (fun i -> Swipe 0.0 400.0)
```

We'll use the same `<div>` trick as before to attach the animation to each flowlet page: since each animation is added as an attribute, it's possible to add the flow one in, too: 

```fsharp
    let AnimateFlow pg =
        Div
            [
                Attr.Style "position" "relative"
                Attr.AnimatedStyle "opacity" FadeTransition (View.Const 1.0) string
                Attr.AnimatedStyle "left" SwipeTransition (View.Const 0.0) (fun x -> (string x) + "px")
            ]

            [ pg ]
```

And finally, all we need to do to animate a flowlet page is to pipe it to the `AnimateFlow` function we just defined:

```fsharp
let personFlowlet =
    Flow.Define (fun cont ->
        let rvName = Var.Create ""
        let rvAddress = Var.Create ""
        Form [cls "form-horizontal" ; "role" ==> "form"] [
            ...
        ]
        |> AnimateFlow
    )
```

...and we're done!


## Finishing Up

Thanks for reading, and I hope I've shown how you can use the declarative animation functionality in UI.Next to animate single-page applications. Of course, I'm only scratching the surface here! Expect some more samples and posts in the coming weeks.

Any questions and comments are very welcome as ever!
