---
title: "Variations for a WebSharper shopping cart"
categories: "f#,websharper,ui,templating,reactive,fsadvent"
abstract: ""
identity: "-1,-1"
---
## Variations for a WebSharper shopping cart - Part I

(This article is part of [F# Advent 2020](https://sergeytihon.com/2020/10/22/f-advent-calendar-in-english-2020/) - a huge shout-out and thanks to Sergey Tihon for organizing it!)

A long time ago, sometime in 2010, I wrote an article about [implementing a shopping cart with WebSharper](/user/granicz/20100930-tutorial-implementing-a-shopping-cart-with-websharper.md). It used **first-class F# events** to implement a reactive shopping cart that updates without page refresh. While you can still read that article today, time has certainly left its mark on it: the code is based on ASP.NET master pages/templates, screenshots are lost, etc. But more importantly, while most of that code would still run today (albeit, with the help of deprecated libraries such as `WebSharper.Html`), things in [WebSharper](https://websharper.com) land have progressed a long ways since then. So I thought I would highlight some of these advancements, bringing the original example back to life.

To follow along and make your own changes, you can grab the code from the following GitHub repo:

[![](https://cdn.iconscout.com/icon/free/png-64/github-169-1174970.png)](https://github.com/websharper-samples/ShoppingCart)

## 1. Start with the design

> WebSharper has long advocated and pioneered the idea of working with **abstract** UIs (formlets, piglets, [reactive forms](https://github.com/dotnet-websharper/forms) - basically, **F# code**) that go hand-in-hand with application logic, to enable developers to **write entire applications without the need to have an actual concrete design up front**, cutting the long dreaded dependency between design and development. Sadly, many developers failed to embrace this approach despite its numerous advantages - which would easily deserve an entire series of articles to highlight, but perhaps more on that later.

So instead, I will stick to the ordinary approach and show how to develop a design template into WebSharper.UI building blocks that will make up the main application UI later. This will include a couple dull HTML preparation steps, so just please bear with me.

First, here is the very same skeleton design from the 2010 article (should be easy to change, just follow the same steps below), and its hideously backwards but dead-simple HTML (don't worry about the CSS, it's in the repo):

[![](https://i.imgur.com/VlLUd4Ul.png)](https://i.imgur.com/VlLUd4U.png)

```html
<html>
<head>
    <link href="reset.css" rel="stylesheet" type="text/css" />
    <link href="site.css" rel="stylesheet" type="text/css" />
    <title>A simple site</title>
</head>
<body>
    <table>
        <tbody>
            <tr>
                <td colspan="2" id="banner">
                    <h1>Your site</h1>
                </td>
            </tr>
            <tr>
                <td id="main">
                    Main content here...
                </td>
                <td id="side">
                    Sidebar content here...
                </td>
            </tr>
            <tr>
                <td colspan="2" id="footer">
                    Your footer goes here...
                </td>
            </tr>
        </tbody>
    </table>
</body>
</html>
```

To prepare this into a fully featured master design, first we add the master placeholders (where the store items and the shopping cart items are displayed, etc.), turning `index.html` into a WebSharper.UI template (you can read about WebSharper.UI and the various reactive and templating primitives [here](https://developers.websharper.com/docs/v4.x/fs/ui)):

```html
<html>
<head>
    <link href="reset.css" rel="stylesheet" type="text/css" />
    <link href="site.css" rel="stylesheet" type="text/css" />
    <title>${Title}</title>
</head>
<body>
    <table>
        <tbody>
            <tr>
                <td colspan="2" id="banner">
                    <h1>${Title}</h1>
                </td>
            </tr>
            <tr>
                <td id="main">
                    <div ws-replace="Main"></div>
                </td>
                <td id="side">
                    <div ws-replace="Sidebar"></div>
                </td>
            </tr>
            <tr>
                <td colspan="2" id="footer">
                    <div ws-replace="Footer"></div>
                </td>
            </tr>
        </tbody>
    </table>
    <script ws-replace="scripts"></script>
</body>
</html>
```

Next, we add the actual design for all the parts we need: items in various categories to the main panel, and a shopping cart in the sidebar. I like to do this inside placeholders, so that the HTML file still renders OK as is, making subsequent refinement (edit and refresh) easier.

For instance, here is our main section, listing various categories of computers:

```html
...
                    <div ws-replace="Main">
                        <div id="shopping-cart">
                            <div class="family">
                                <h1>Laptops</h1>
                                <div class="product">
                                    <img alt="Toshiba" src="/images/laptop.png" />
                                    <div>
                                        <h1>Toshiba</h1>
                                        <p>
                                            <code>$1299 / item</code>
                                        </p>
                                        <input type="text" value="1" />
                                        <button>Add to cart</button>
                                    </div>
                                </div>
                                <div class="product">
                                    <img alt="HP" src="/images/laptop.png" />
                                    <div>
                                        <h1>HP</h1>
                                        <p>
                                            <code>$1499 / item</code>
                                        </p>
                                        <input type="text" value="1" />
                                        <button>Add to cart</button>
                                    </div>
                                </div>
                                ...
                            </div>
                            <div class="family">
                                ...
                            </div>
                            ...
                        </div>
                    </div>
```

And this is what it looks like when it's turned into a set of inner templates and additional placeholders:

```html
                    <div ws-replace="Main">
                        <div ws-template-children="ShoppingCart">
                            <div id="shopping-cart">
                                <div ws-replace="Families">
                                    <div ws-template-children="Family">
                                        <div class="family">
                                            <h1>${FamilyTitle}</h1>
                                            <div ws-replace="Products">
                                                <div ws-template-children="Product">
                                                    <div class="product">
                                                        <img alt="${ProductTitle}" src="${ProductImageSrc}" />
                                                        <div>
                                                            <h1>${ProductTitle}</h1>
                                                            <p>
                                                                <code>$ ${ProductPrice} / item</code>
                                                            </p>
                                                            <input type="text" value="1" />
                                                            <button ws-onclick="AddToCart">Add to cart</button>
                                                        </div>
                                                    </div>
                                                </div>
                                            </div>
                                            <div class="product">
                                                <img alt="HP" src="/images/laptop.png" />
                                                <div>
                                                    <h1>HP</h1>
                                                    <p>
                                                        <code>$1499 / item</code>
                                                    </p>
                                                    <input type="text" value="1" />
                                                    <button>Add to cart</button>
                                                </div>
                                            </div>
                                            ...
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
```

At this point, our file renders "almost nicely", with the exception of the inner templates (for items and categories, and the shopping cart) we introduced:

| Original | WebSharper.UI |
| - | - |
|[![](https://i.imgur.com/KtPJxi8l.png)](https://i.imgur.com/KtPJxi8.png) | [![](https://i.imgur.com/w47y2v7l.png)](https://i.imgur.com/w47y2v7.png) |

## 2. Create a new WebSharper SPA on top of the design

Now that we have a basic design and sprinkled in the templating primitives (`ws-replace` placeholders for content, `ws-onclick` event handlers, and named our inner templates), it's time to put our F# code in place. For this application, instead of the usual client-server WebSharper application template, we will use the WebSharper SPA one for simplicity. In a real-life shopping cart, you will inevitably need a server or backend service to post to; this you can add later pretty easily.

I normally just do the following:

0) Make sure I have the basics first:

    a) The latest `dotnet` CLI. The easiest way for that is to [install the latest .NET Core SDK](https://docs.microsoft.com/en-us/dotnet/core/install/windows?tabs=net50).

    b) The latest WebSharper project templates. You can grab them with:

        dotnet new -i WebSharper.Templates
    
1) Create a WebSharper SPA:

        dotnet new websharper-spa -lang f# -n MyShoppingCart

2) Copy the design template `index.html` to the `wwwroot` folder, overwriting the file there, with the following additions:

    a) Add the following HTML, say, to the `<head>` section, to remove the initial flickering between the document load event and the time WebSharper.UI's bind event fires (which hides all templates in the document and binds the placeholders):

        <style>
            /* Don't show the not-yet-loaded templates */
            [ws-template], [ws-children-template] { display: none; }
        </style>


    b) Add the following HTML just before the end of the `<body>` tag, to load the generated JavaScript for the app we will be developing (note the filename matching the project folder/name we created earlier):

        <script type="text/javascript" src="Content/MyShoppingCart.min.js"></script>

3) At this point, we are ready to change `Client.fs` and get rid of those compiler errors.


## 3.a. Write the application logic - Approach #1 - Reactive UI

Here is our entire app:

```fsharp
namespace MyShoppingCart

open WebSharper
open WebSharper.JavaScript
open WebSharper.UI
open WebSharper.UI.Client
open WebSharper.UI.Templating
open WebSharper.UI.Notation

[<JavaScript>]
module DTO =
    type Product =
        {
            Id: string
            Title: string
            Price: int
            ImageSrc: string
        }

    type ProductFamily =
        {
            Title: string
            Products: Product list
        }
    
    type Store = ProductFamily list

[<JavaScript>]
module Client =
    // The templates are loaded from the DOM, so you just can edit index.html
    // and refresh your browser, no need to recompile unless you add or remove holes.
    type IndexTemplate = Template<"wwwroot/index.html", ClientLoad.FromDocument>

    type CartItem =
        {
            Item: Product
            Quantity: int
        }

    // Our cart is a reactive list - changes to it
    // are immediately reflected on the UI.
    type Cart = ListModel<string, CartItem>

    // Populate our store. This can be changed to fetch from a
    // server if needed.
    let store =
        let item imageSrc (title, id, price) =
            {
                Id = id
                Title = title
                Price = price
                ImageSrc = imageSrc
            }
        let laptop product = item "/images/laptop.png" product
        let desktop product = item "/images/desktop.png" product
        let netbook product = item "/images/netbook.png" product
        [
            {
                Title = "Laptops"
                Products =
                    [
                        laptop ("Toshiba", "id1", 1299)
                        laptop ("HP", "id2", 1499)
                        laptop ("Dell", "id3", 1499)
                        laptop ("Acer", "id4", 1499)
                    ]
            }
            {
                Title = "Laptops"
                Products =
                    [
                        desktop ("Gamer 1", "id11", 699)
                        desktop ("Gamer 2", "id12", 799)
                        desktop ("Office", "id13", 599)
                        desktop ("Server", "id14", 1299)
                    ]
            }
            {
                Title = "Laptops"
                Products =
                    [
                        netbook ("Entry", "id21", 799)
                        netbook ("Medium", "id22", 899)
                        netbook ("Cool", "id23", 699)
                        netbook ("Speed-King", "id24", 999)
                    ]
            }
        ]

    // Set up empty cart
    let cart : Cart = ListModel.Create (fun item -> item.Item.Id) []

    [<SPAEntryPoint>]
    let Main () =
        IndexTemplate()
            .Title("My Shop")
            .Footer("MyShoppingCart - a simple WebSharper.UI demo app")
            .Main(
                IndexTemplate.ItemsToSell()
                    .Families(
                        store
                        |> List.map (fun family ->
                            IndexTemplate.Family()
                                .Title("Laptops")
                                .Products(
                                    family.Products
                                    |> List.map (fun product ->
                                        IndexTemplate.Product()
                                            .Title(product.Title)
                                            .Price(string product.Price)
                                            .ImageSrc(product.ImageSrc)
                                            .Quantity(string 1)
                                            .AddToCart(fun e ->
                                                if cart.ContainsKey(product.Id) then
                                                    let item = cart.Lens(product.Id)
                                                    let quantity = item.LensAuto (fun item -> item.Quantity)
                                                    quantity := quantity.Value + int e.Vars.Quantity.Value
                                                elif int e.Vars.Quantity.Value <> 0 then
                                                    cart.Add
                                                        {
                                                            Item = product
                                                            Quantity = int e.Vars.Quantity.Value
                                                        }
                                                else ()
                                            )
                                            .Doc()
                                    )
                                )
                                .Doc()
                        )
                    )
                    .Doc()
            )
            .Sidebar(
                IndexTemplate.ShoppingCart()
                    .CartItems(
                        cart.View.DocSeqCached (fun item ->
                            IndexTemplate.CartItem()
                                .Name(item.Item.Title)
                                .Quantity(string item.Quantity)
                                .Amount(string (item.Quantity * item.Item.Price))
                                .Remove(fun e ->
                                    cart.RemoveByKey(item.Item.Id)
                                )
                                .Increment(fun e ->
                                    cart.Lens(item.Item.Id) := { item with Quantity=item.Quantity+1 }
                                )
                                .Decrement(fun e ->
                                    if item.Quantity <= 1 then
                                        cart.RemoveByKey(item.Item.Id)
                                    else
                                        cart.Lens(item.Item.Id) := { item with Quantity=item.Quantity-1 }
                                )
                                .Doc()
                        )
                    )
                    .TotalAmount(
                        cart.View
                        |> View.Map (Seq.sumBy (fun item -> item.Item.Price * item.Quantity))
                        |> View.Map string
                    )
                    .Checkout(fun _ -> JS.Alert("Checkout was called!"))
                    .EmptyCart(fun _ ->
                        cart.Clear()
                    )
                    .Doc()
            )
            .Bind()
```

A couple bits worth noting:

 * We see our design template placeholders and inner templates under `IndexTemplate`, which is a type space brought in through the WebSharper.UI templating type provider:

    ```fsharp
     // The templates are loaded from the DOM, so you just can edit index.html
    // and refresh your browser, no need to recompile unless you add or remove holes.
    type IndexTemplate = Template<"wwwroot/index.html", ClientLoad.FromDocument>
    ```

    > As the code comment says, one of the nicest things about this setup, apart from **never missing a placeholder** and the **type safety and perfect matching of design to code** we all enjoy so much, is that any design changes you make to the HTML file are **immediately reflected** (aka **hot reload**) in your app, all you need is to refresh your page in the browser. This makes WebSharper templating far superior to any "embed your HTML into F# code" approach, which of course is still available if/when you need it. (You will only need to recompile if you add/remove placeholders or event handlers.)

 * The products in the store are a given: data is populated on the client. If you want to fetch this info from the server, you can move that block to an RPC function (say, in a separate module to make your code clearer):

    ```fsharp
    module Server =
        open DTO

        [<Rpc>]
        let GetStoreInfo () =
            ...
    ```

    At this point, you will get a WebSharper warning about non-asynchronous server calls, so just wrap the above RPC function into:

    ```fsharp
            |> async.Return
    ```

    Then in your client code, call it appropriately:

    ```fsharp
    let store =
        async {
            return! Server.GetStoreInfo()    
        }
        |> Async.RunSynchronously
    ```

 * The main panel is populated by mapping each product family and the products inside it into markup (represented by the `Doc` type). We don't expect any changes to our product hierarchy, but if there were any, we would need to make that mapping reactive.

 * The shopping cart on the other hand is bound **reactively**. Since our data model is composite/aggregate and not a single atomic value, we use a `ListModel`, which is a built-in reactive store for a list of values (of any shape, including further aggregates.)

    A WebSharper.UI reactive model is based on reactive variables and their current values (we call these "views".) To introduce any UI reactivity, we need to turn these reactive views into reactive markup. For a list model, we can do that with `DocSeqCached`, which takes a function from the underlying, individual (non-reactive) values to markup (`Doc`), and creates an aggregated markup block that will react to changes inside those values (single or composite). Here, we don't deal with two-way binding, i.e. changes originating from the UI back to the data model, but that's similarly easy to set up (via `ws-var` in the markup.)

    When you dig deeper in the code and the design template, you will see that I marked the [+], [-], [x] buttons in the shopping cart widget with a special `ws-onclick="..."` attribute. This makes those click event handlers available to bind in F# code. In those event handlers, we make use of WebSharper.UI lenses to access a particular cart item as a reactive variable, and make changes to it through that reference. (You can read about the [V notation](https://developers.websharper.com/docs/v4.x/fs/ui#v) as well for an alternative syntax.)

    ```fsharp
    cart.View.DocSeqCached (fun item ->
        IndexTemplate.CartItem()
            .Name(item.Item.Title)
            .Quantity(string item.Quantity)
            .Amount(string (item.Quantity * item.Item.Price))
            .Remove(fun e ->
                cart.RemoveByKey(item.Item.Id)
            )
            .Increment(fun e ->
                cart.Lens(item.Item.Id) := { item with Quantity=item.Quantity+1 }
            )
            .Decrement(fun e ->
                if item.Quantity <= 1 then
                    cart.RemoveByKey(item.Item.Id)
                else
                    cart.Lens(item.Item.Id) := { item with Quantity=item.Quantity-1 }
            )
            .Doc()
    ``` 

 * The "Add to cart" button knows the identity of each product from its closure, and we grab a reactive reference to an existing cart item through a lense, so we can update its quantity:

    ```fsharp
    family.Products
    |> List.map (fun product ->
        IndexTemplate.Product()
            // ...
            .AddToCart(fun e ->
                if cart.ContainsKey(product.Id) then
                    let item = cart.Lens(product.Id)
                    let quantity = item.LensAuto (fun item -> item.Quantity)
                    quantity := quantity.Value + int e.Vars.Quantity.Value
                elif int e.Vars.Quantity.Value <> 0 then
                    cart.Add
                        {
                            Item = product
                            Quantity = int e.Vars.Quantity.Value
                        }
                else ()
            )
    ```

    Another key concept to note is the use of `e.Vars`. Here, `e` is the target of the event handler, which usefully "knows" about the data model of its surrounding template. In this case, it knows that there is a `Quantity` field that is two-way bound to the quantity input box on the UI (since we made that binding explicit with `ws-var="..."`), and we can read its current value with `.Value`.

    All in all, data/model changes to the shopping cart (via `cart.Add` for adding, `cart.Remove` for removing, and `cart.Lens(...) := ...` for updating) reflect on the UI immediately, and only those DOM nodes that need to be changed are indeed changed. You can see this in action below:

    ![](https://i.imgur.com/5eKJs0J.gif)

    WebSharper.UI's dataflow algorithm, detecting model changes and applying/reflecting them to the UI and back is stunningly fast. In our earlier benchmarks, we observed a 10-20% performance improvement over using hand-written React, which is quite a feat considering that we didn't have to write a single line of JavaScript.

## 3.b. Write the application logic - Approach #2 - WebSharper.Mvu

I thought it would be a fun experiment to contrast the mutable, reactive WebSharper.UI implementation above with an immutable WebSharper.Mvu implementation. The latter is not a 100% Elm-style MVU but a slightly modified one, with the couple important differences and benefits. I plan to describe these in a follow-up article.

## 4. Conclusions

WebSharper.UI templating and reactive data models are an effective way to develop highly performant web applications. The [original 2010 article](/user/granicz/20100930-tutorial-implementing-a-shopping-cart-with-websharper.md) used first-class F# events and WebSharper HTML-in-F# components, which altogether was significantly more code and didn't offer a streamlined developer experience. The revisited implementation above makes HTML changes immediate with a simple page refresh, frees us from having to write F# for HTML, provides strong guarantees about templating type safety, and gives a ultra-performant two-way binding for reactive components.

Now, there are still a number of things that can be improved or streamlined further. If you are interested in discovering more, come and chat in the [WebSharper Gitter channel](http://gitter.im/intellifactory/websharper):

[![](https://badgen.net/gitter/members/intellifactory/websharper)](http://gitter.im/intellifactory/websharper)

Happy holidays!
