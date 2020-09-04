---
title: "WebSharper.UI.Next: An Introduction"
categories: "dataflow,ui.next,dom,web,f#,websharper"
abstract: "In this blog, we introduce an experimental UI library for the WebSharper web framework, and illustrate this by means of a small example. "
identity: "3954,77293"
---
Here at [IntelliFactory](http://www.intellifactory.com), we've been working on a new way of constructing single-page applications.

We're using [WebSharper](http://www.websharper.com), which is a framework for writing web applications entirely in F#, without the need to write separate JavaScript, PHP and SQL code: if you haven't used it before, I encourage you to have a play around!

The main idea behind this library is centered around data sources. These data sources, which we call `Var`s or *reactive variables*, can be linked to form components, or set by remote server calls, for example. We then have *views*, which provide a way of observing these changing variables, which then can be embedded directly into the DOM in such a way that they automatically update whenever the data source changes. On top of this, we use a functional style, and take steps in the dataflow layer to ensure that memory leaks don't occur.

Firstly, if you want to see everything in action, you can find a live examples site (itself written using the framework, of course!), along with the source code for the examples, on [GitHub](https://websharper-samples.github.io/ui/).

Briefly, WebSharper.UI.Next consists of two layers: a dataflow layer, made up of variables and views, and a reactive DOM layer making use of this, allowing time-varying elements to be embedded in the DOM. In the simplest case, it's possible to create a reactive variable (which can be thought of as a *data source*), bind this to a DOM element, and have the DOM element update automatically whenever the data changes. This removes the need to write separate DOM handling code for whenever the data changes: you just specify the view once, and the updates happen automatically.

In this sense, the library is a bit like libraries such as [Facebook React](http://facebook.github.io/react/). Firstly though, everything's in F#, so we get all of the advantages of a strong, static type system, including the ability to catch errors sooner and make use of such a type system for structuring applications. It's also easier for us to make use of traditional, powerful functional abstractions when writing code. React is built on a virtual DOM system, which efficiently propagates updates via a diff algorithm against the "real" DOM. While we do some diff'ing, too, this is purely an optimisation step: we update the DOM by re-linking from the desired position, much as we do in the Formlets library for WebSharper.

## An example

Perhaps the best way to demonstrate the library is via an example. Say we have an input box, and we want to use this as an input to multiple different functions, each doing something different with the data. In this case, it should be possible to write something in a box, and then display the entered text, the input text but capitalised, the input text but reversed, the number of words, and whether the word count is odd or even. 

Let's dive right in. The actual code for this example contains a bit more styling than I have here, so I'm just going to talk about the interesting bits. Firstly, we create a reactive variable to store the input text. 

```fsharp
let rvText = Var.Create ""
```

This creates a variable `rvText` of type `Var<string>`, which is essentially a 'cell' containing a string. 

The next step is to create an input box, which is linked with this variable. Whenever a user types into this box, the variable will be updated, and anything which depends on it will be updated automatically. In addition to this, if something else updates the variable, the text in the box will be updated accordingly. Achieving this is simple:

```fsharp
Doc.Input [] rvText
```

(aside: the empty list parameter specifies attributes, which may themselves be backed by reactive variables).

Now we have the data input part, we need to have a look at how to display it. In order to do this, we create a reactive *view*. Views can be thought of a way to "peer into" a variable as it varies with time, and it's views that we attach to the DOM. It's also worth saying that views are, as their name would suggest, read-only. We can also chain different views together using applicative and monadic combinators, allowing a dynamic dataflow graph to be constructed. 

Constructing a view from a variable is simple:

```fsharp
let view = View.FromVar rvText
```

The `view` variable then has the type `View<string>`. We can then embed this into the DOM by creating a text node inside a `<div>` tag:

```fsharp
Doc.Element "div" [] [
    Doc.TextView view
]
```

With this view, it's then possible to create the other views we want by using the `View.Map : ('A -> 'B) -> View<'A> -> View<'B>` function. `Map` simply creates a new view, with the function applied. Using this, we can easily implement all of the other ways of looking at the input:

```fsharp
let viewCaps = 
    view |> View.Map (fun s -> s.ToUpper () )

let viewReverse = 
    view |> View.Map (fun s -> new string ((s.ToCharArray ()) |> Array.rev))

let viewWordCount = 
    view |> View.Map (fun s -> s.Split([| ' ' |]).Length)

let viewWordCountStr = 
    View.Map string viewWordCount

let viewWordOddEven = 
    View.Map (fun i -> if i % 2 = 0 then "Even" else "Odd") viewWordCount
```

...and these can simply be embedded into the DOM as before.

## What's next?

We think that the experiment so far has gone quite well, and that we've demonstrated that this sort of dataflow structure works nicely with WebSharper and the DOM. Building on this, we're currently working on two different things: adding support for animation, and incorporating and using this framework to build on existing functional web abstractions.

For the animation side of things, we're hoping to find an intuitive, idiomatic, but powerful way to declaratively encode animations: we're looking at D3 for inspiration, but hope to improve the interface somewhat! This should mean that it's possible to use the dataflow layer to drive powerful visual, interactive documents in a pleasant, functional manner.

The second thing we're trying to do is to use UI.Next alongside different functional abstractions, making it easier to create interactive web applications. We've done some work on porting flowlets to the new system, which are an extension of formlets with a monadic binding operation to allow for dynamic composition. In particular, this means that it's possible to setout a "flow" for an application, showing how data from one stage of an application is used in a different part. More details are to come in a later blog post, but you can see the code for one example [here](https://github.com/websharper-samples/ui/blob/master/src/ContactFlow.fs).

In particular, the main part of this example is as follows:

```fsharp
let exampleFlow =
    Flow.Do {
        let! person = personFlowlet
        let! ct = contactTypeFlowlet
        let! contactDetails = contactFlowlet ct
        return! Flow.Static (finalPage person contactDetails)
    }
    |> Flow.Embed
```

We firstly get some user information from the `personFlowlet` page, determine which type of contact information to get using the `contactTypeFlowlet` page, use this to get the required information, and finally display all of these on the final, static page. 

In addition to this, we're hoping to see how this approach integrates with Piglets, our library for pluggable UIs.

Thanks for reading, and stay tuned for more! If you have any comments, we'd love to hear them.
