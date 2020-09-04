---
title: "Structuring Web Applications with WebSharper.UI.Next"
categories: "reactivedom,spa,ui.next,f#,websharper"
abstract: "In this post, we see how WebSharper.UI.Next may be used to structure single-page applications with multiple sub-pages, and how this can be integrated with URL synchronisation."
identity: "3965,77308"
---
Since our [last post](http://www.websharper.com/blog-entry/3954), we have been hard at work extending the basic functionality of WebSharper.UI.Next to include some higher-level things, in particular support for animation, and some functional abstractions which make creating single-page applications (SPAs) safer, easier, and more intuitive. In this post, we'll talk a bit about some of the web abstractions we've added; [Anton](http://fpish.net/profile/anton.tayanovskyy) has written about the animation functionality [here](http://websharper.com/blog-entry/3964).

For the uninitiated, WebSharper.UI.Next is our upcoming framework for creating rich single-page applications. By creating reactive variables, and views which allow these variables to be observed as and when they change, it is possible to create highly-reactive applications backed by a purely-functional model. You can see some examples of this framework at work [here](http://intellifactory.github.io/websharper.ui.next), and the GitHub repository showing the source code for the website [here](http://www.github.com/intellifactory/websharper.ui.next).

A key advantage of using WebSharper to develop applications is the use of F#'s type system not only to provide better safety guarantees --- that is, fewer runtime errors and less JavaScript debugging --- but also to help better structure applications. To this end, IntelliFactory have made use of approaches such as [Formlets](http://www.ezrakilty.net/pubs/formlets.pdf) to better structure handling user input, and developed our own such as [Flowlets](http://link.springer.com/chapter/10.1007%2F978-3-642-24276-2_13#page-1) and [Piglets](http://www.websharper.com/docs/piglets). This past week, we've been working on seeing how some of these fit with SPA applications, and showing how they can be used within UI.Next. In particular, this past week we have looked at Flowlets, which help with structuring parts of an application which have a linear control flow, and how UI.Next can help to structure sites without such a control flow. Additionally, we've looked at how it's possible to synchronise the current position in a single-page application with the URL bar, to make sharing pages easier.

## Flowlets

Flowlets are built to make it easier to structure applications (or indeed segments of an application: everything is composable) which have a linear control flow. By way of example, think for example of a registration form which has multiple pages, each getting different pieces of information, where each page may depend on information provided by a previous page. 

In our small example, we first ask the user for some common information: in this case, a name and address. After this information is provided, we then ask the user whether they wish to specify a name or a phone number. Depending on which option is selected, we then display a different form to retreive the required information, before displaying the results on a final page. You can find this live [here](https://websharper-samples.github.io/ui/#/samples/ContactFlow), and the source code [here](https://github.com/websharper-samples/ui/blob/master/src/ContactFlow.fs). The source code contains a bit of extra markup, but we'll concentrate on the main specifics here instead.

Firstly, we define our model, containing the data which we wish to retrieve. A key advantage of using F# for this kind of thing is that we can work directly with a model we have created in F#, without having to think about JavaScript objects and such. 

```fsharp
type Person =
    {
        Name : string
        Address : string
    }

type ContactType = | EmailTy | PhoneTy

type ContactDetails =
    | Email of string
    | PhoneNumber of string
```

The `Person` type consists of a name and address. The `ContactType` type specifies whether the user has opted to specify an e-mail address or a phone number, and the `ContactDetails` discriminated union specifies the data which has been submitted. 

Now, with the model created, we can look at how everything fits together. 

To create a flowlet, we use the `Define` function.

```fsharp
val Define : (('A -> unit) -> Doc) -> Flow<'A> 
```

Here, `(('A -> unit) -> Doc)` is a function which takes a callback of type `('A -> unit)`, which is used to progress to the next stage of the flowlet, and produces a `Doc`. For readers unfamiliar with `Doc`, this is our monoidally-defined representation of a fragment of a reactive DOM tree, used in this case to render the flowlet. This in turn produces a value of type `Flow<'A>`, where `'A` is the type of data returned by that flowlet page.

Now, we create some pages. As I say, I've stripped out the markup, concentrating on the bits of the code we care about.

Firstly, here's the first page which grabs the name and address from the user.

```fsharp
let personFlowlet =
    Flow.Define (fun cont ->
        let rvName = Var.Create ""
        let rvAddress = Var.Create ""

        Doc.Input [] rvName 
        Doc.Input [] rvAddress 
        Doc.Button "Next" [cls "btn" ; cls "btn-default"] (fun () ->
            let name = Var.Get rvName
            let addr = Var.Get rvAddress
            // We use the continuation function to return the
            // data we retrieved from the form.
            cont ({Name = name ; Address = addr})
        )
    )
```

The `personFlowlet` function is of type `Flow<Person>`: a flowlet segment returning a value of type `Person`. We begin the function with `Flow.Define`, which takes a continuation function `cont`. The next thing to do is specify reactive variables for the name and address fields, initialised with the empty string. 

Once we have the variables, we can use these with the `Doc.Input` function to create a form control which populates the variables. The submission button is then created using a function which gets the current calues of the name and address fields, and then passes a `Person` value to the continuation function.

Using the same technique, we can then make three further pages: `contactTypeFlowlet`, which returns a value of type `ContactType`; `contactFlowlet`, which takes a `ContactType` and returns a `ContactDetails` value, and finally a static page which displays the details. 

After this is complete, all that is left is to tie everything together. Making use of the fact that flowlets have a monadic interface, we can construct flowlets using a computation expression. By doing this, we specify the order in which the pages appear, and how the data collected from one page may be used in another. 

```fsharp
let PersonContactFlowlet =
    Flow.Do {
        let! person = personFlowlet
        let! ct = contactTypeFlowlet
        let! contactDetails = contactFlowlet ct
        return! Flow.Static (finalPage person contactDetails)
    }
    |> Flow.Embed
```

In this computation expression, `preson` is of type `Person`, `ct` is of type `contactTypeFlowlet`, and `contactDetails` is of type `ContactDetails`. It's also clear that the `contactFlowlet` flowlet depends on the value retrieved from the `contactTypeFlowlet` flowlet page. Finally, we return a static page displaying the details, and then use the `Flow.Embed` combinator to embed the flowlet into a reactive DOM fragment of type `Doc`, which can be composed as normal.

### Structuring Non-Linear Sites

Flowlets are great for specifying parts of an application which have a linear control flow. They don't work so well when we want to jump around freely between different parts, however. 

Using UI.Next, it's also easy to construct SPA applications consisting of multiple 'pages', which are loaded in a structured manner. In particular, by using UI.Next, everything can be specified without thinking about URLs at all, but purely passing the page we want to load as a continuation function. It's best to hop straight in with an example. Let's make a small, silly website.

To start with, we need to create a discriminated union representing each page in the site. 

```fsharp
type Page =
    | BobsleighHome
    | BobsleighHistory
    | BobsleighGovernance
    | BobsleighTeam
```

By creating the `Page` type, we'll then be able to create a `Var<Page>` which specifies the current page we're on. We'll consider two different types of navigation: *global* navigation performed by a navigation bar, and *local* navigation, performed by links on individual pages.

Now, we'll make implementations of each page. Each page will take a record, `Context` containing a continuation function, which can be used to change the current page. For example, here is the source for the home page:

```fsharp
    type Context = {
        Go : Page -> unit
    }

    let HomePage ctx =
        Doc.Concat [
            el "div" [
                el "h1" [txt "Welcome!"]
                el "p" [
                    txt "Welcome to the IntelliFactory Bobsleigh MiniSite! Here you can find out about the "
                    link "history" [] (fun () -> ctx.Go BobsleighHistory)
                    txt " of bobsleighs, the "
                    link "International Bobsleigh and Skeleton Federation" [] (fun () -> ctx.Go BobsleighGovernance)
                    txt ", which serve as the governing body for the sport, and finally the world-famous "
                    link "IntelliFactory Bobsleigh Team." [] (fun () -> ctx.Go BobsleighTeam)
                ]
            ]
        ]
```

You'll notice here that each link is associated with a callback, which executes the `ctx.Go` function, providing the next page to show.

Tying this all together, we get a main function which looks like this:

```fsharp
let Main () =
    let m = Var.Create BobsleighHome

    let withNavbar =
        Doc.Append (NavBar m)

    let ctx = {Go = Var.Set m}
    View.FromVar m
    |> View.Map (fun pg ->
        match pg with
        | BobsleighHome -> HomePage ctx |> withNavbar
        | BobsleighHistory -> History ctx |> withNavbar
        | BobsleighGovernance -> Governance ctx |> withNavbar
        | BobsleighTeam -> Team ctx |> withNavbar)
    |> Doc.EmbedView
```

Here, we create a context instance with `Var.Set m` as the continuation function. We then create a view of the page variable, which we can use to update the display of the page whenever it changes. In particular, through the use of `View.Map`, we change a `View<Page>` and create a `View<Doc>`. This can then be embedded into the reactive DOM using the `Doc.EmbedView : View<Doc> -> Doc` combinator. 

Global actions, for example if we have a shared component such as a navigation bar, can be created by passing the page variable as a parameter. To change the current page, it's just necessary to set the variable. 

### URL Routing

Finally, SPA applications often lose the ability to do such things as using the 'back' button to go back in the application, or use the URL to share the current point in the application. We've worked on this to create a "Site" abstraction to allow different parts of the site to be responsible for different parts of the URL: the key here is that these sites are *composable*, meaning that it is possible to have different sub-sites.

The key idea behind this is the `Router<'T>` type, which essentially specifies a bijection between fragments of the URL, and page representations such as `Page` in the site above. If we were to want to extend the bobsleigh mini-site with some kind of URL routing, a router could be created as follows:

```fsharp
    let TheRouter =
        Router.Create
            (function
                | BobsleighHome -> Route.Create []
                | BobsleighHistory -> Route.Create [RouteFrag.Create "history"]
                | BobsleighGovernance -> Route.Create [RouteFrag.Create "governance"]
                | BobsleighTeam -> Route.Create [RouteFrag.Create "team"])
            (fun route ->
                match Route.ToStringList route with
                | [] -> BobsleighHome
                | ["history"] -> BobsleighHistory
                | ["governance"] -> BobsleighGovernance
                | ["team"] -> BobsleighTeam
                | xs -> BobsleighHome)
```

We could then use the resulting router to create a `Site`, which is then merged with other `Site`s, to create a routing trie. If you want to see this in action, check out [the live example](https://websharper-samples.github.io/ui/#/samples/RoutedBobsleighSite), and you can also take a look at the source [here](https://github.com/websharper-samples/ui/blob/master/src/RoutedBobsleighSite.fs). Indeed we use this technique on our main samples site itself!


### The Future

Our goals now are to get some more functional abstractions added in, particularly those used for handling user input such as forms. In particular, it would be nice to see how we can integrate Loic Denuziere's work on Piglets to make use of the reactive backend, instead of using the stream abstraction. Additionally, we're adding features and examples all the time, so keep checking back!
