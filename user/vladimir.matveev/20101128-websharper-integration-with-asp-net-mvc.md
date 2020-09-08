---
title: "WebSharper: Integration with ASP.NET MVC"
categories: "f#,websharper,aspnet,mvc,web"
abstract: "ASP.NET MVC framework is lightweight web framework with strong emphasis on loose coupling and testability of involved components. This post will be dedicated to the major subject: how to create WebSharper web application based on ASP.NET MVC so it can take the best  from both counterparties."
identity: "1039,74620"
---
With MVC being a server-centric framework and WebSharper being the #1 platform for creating client-based applications, marrying the two can yield a strong synergy. In the example I am developing here, MVC is applied for the server-side code, and rich client-side capabilities are rendered by WebSharper. So let’s jump straight into it…First, our web application solution will consist of two Visual Studio projects: a C# web application that contains the MVC machinery such as views and presentation style, and an F#/WebSharper project that powers the client-side functionality:

### Model

In MVC, models contain core information about the application, such as domain-specific logic, etc. For the sake of brevity, the Model component in our demo will be very simple and represented by some data without any particular behavior.

```fsharp
type Item = {
    Id : int
    Name : string
    Description : string
    }
 
type AppModel = {
    Items : ResizeArray<Item>
    }
```

### ViewModel

MVC aims to separate the responsibilities between the different parts of an application, and advises that the View component is not to contain any business or data access logic, but instead simply to act as a renderer that formats its output based on some simple pieces of data/input – the view model. In our application this view model has almost the same structure as the Model component above, even though in most real-world application the View model can be quite sophisticated.

```fsharp
type ViewModel = {
    Items : Item array
    }
```

### Controller

Controllers are responsible for handling user interaction, working with model and selecting proper view for displaying response.

```fsharp
type PageletsController(model : AppModel) = 
    inherit Controller()
 
    member this.Items() =
        let viewModel = { Items = Array.ofSeq(model.Items) } 
        this.View(viewModel)
```

Our controller here just converts the existing model to the view model, and returns it as the view to be rendered. By default this is `Pagelets\Items.aspx`.

### View

View is basically just a template for outputting an HTML representation. Usually View is mixture of ASPX markup + some JavaScript. However, since we are using WebSharper we will move all client-side logic to a WebSharper component that will be simply embedded in View as a plain `Web.Control` instance.

```xml
<%@ Page Title="" 
           Language="C#" 
           MasterPageFile="~/Views/Shared/Site.Master" 
           Inherits="WebSharperMvcProject.Helpers.DataBoundViewPage" %>
 
<asp:Content ID="Content1" ContentPlaceHolderID="TitleContent" runat="server">
    Pagelets sample
</asp:Content>
<asp:Content ID="Content2" ContentPlaceHolderID="MainContent" runat="server">
    <ws:PageletsControl ID="pagelets" runat="server" Items="<%# Model %>" />
</asp:Content>
```

You might have noticed that we inherit this page not from `ViewPage` but from `DataBoundPage`: this custom base class simply calls `DataBind` inside `OnLoad` to bind a data source to a page controls - thus binding ViewModel to the WebSharper web control. On the 'Render' step ViewModel will be serialized and passed to the client to be used during the actual construction of DOM elements.

### WebSharper Control

```fsharp
type PageletsControl() =
    inherit Web.Control()
 
    [<DefaultValue>]
    val mutable Items :  ViewModel
 
    [<JavaScript>]
    override this.Body = upcast (Pagelets.main this.Items)
```

`PageletsControl` consists of two parts: the first one displays elements from ViewModel as a table, the second one adds a new element to the server-side collection.

This first part demonstrates the basic client-side abilities of WebSharper: HTML combinators, accessing elements by reference instead of locating them by string identifiers, events handling and wiring, and using jQuery natively – with the F# compiler guaranteeing the soundness and type safety of all operations

```fsharp
    [<JavaScript>]
    let main (model : ViewModel) = 
        let itemToAmount = new Dictionary<int, int>()
        let createItemRow (item : Item) = 
            
            let amount = Div [Text ""]
            
            let rec plus = 
                Button [Text "+"]
                |>! OnClick (fun _ _ ->
                    let oldValue = itemToAmount.[item.Id]
                    setAmount (oldValue + 1)
                    )
            and minus = 
                Button [Text "-"]
                |>! OnClick (fun _ _ ->
                    let oldValue = itemToAmount.[item.Id]
                    setAmount (oldValue - 1)
                    )
            and setAmount v = 
                if v = 0 
                then
                    minus.SetAttribute("disabled", "true")
                else    
                    minus.RemoveAttribute("disabled")
                itemToAmount.[item.Id] <- v
                amount.Text <- v.ToString()
            
            setAmount 0

            TR [
                TD [Text item.Name]
                TD [Text item.Description]
                TD [plus]
                TD [minus]
                TD [amount]
            ]

        let items = 
            Table [Border "1"] -< [
                yield TR [ 
                    TD [Attr.Class "tableCaption"] -< [Text "Name"] 
                    TD [Attr.Class "tableCaption"] -< [Text "Description"]; 
                    TD []; 
                    TD []; 
                    TD [Attr.Class "tableCaption"] -< [ Text "Amount" ] 
                    ]
                for item in model.Items do
                    yield createItemRow item
            ]
        
        let statusBar = Div []
        let setStatus text success = 
            statusBar.Text <- text
            let statusClass = if success then "okStatus" else "errorStatus"
            statusBar.AddClass(statusClass)
            let jq = JQuery.JQuery.Of(statusBar.Dom)
            jq.FadeIn(800.0, fun () ->
                jq.FadeOut(800.0, fun () -> 
                    statusBar.RemoveClass(statusClass)
                    ) |> ignore
                ) |> ignore
        
        let addNewItem = // will be described later
                
        Table [
            TR [
                TD [ FieldSet [ Legend [ Text "Select items" ]; items] ]
                TD [ VAlign "top"] -< [ addNewItem; upcast statusBar ]
            ]
        ]
```

### Instance-level RPC

`addNewItem` demonstrates a new feature introduced in WebSharper 2.0, but first – a short point of reference. A common answer to the question "How to send a request to the server without reloading the entire page" is "via an Ajax request." Indeed, libraries such as jQuery provide a convenient way for making this call – for instance, `jQuery.ajax()` + a few of handy helpers (`get`, `post`)... But all these are all far from perfect from the WebSharper point of view – for instance, the compiler cannot verify the types of the arguments, nor the URL of the request. To overcome this issue WebSharper 1.0 provides RPC functions: server-side functions that are accessible from the client with a simple call.

```fsharp
module Rpc = 
    [<Rpc>]
    let someFunction (a : int * string) = …

// somewhere in the client code
Div [...] |>! OnClick (fun _ _ -> Rpc.someFunction (1, "1"))
```

However, these are still far from the ideal – RPC functions are static from the OO perspective with all of their consequences - references can only be made to static data, poor lifetime management, poor testability, and so on. But luckily, WebSharper RPC functions are not the last step of evolution. Without further ado, I’m going to introduce **Instance-level RPC** functions, a new feature in the upcoming WebSharper 2.0 release. On the client-side special function `Remote : T` acts as the access point for the handler instance.

```fsharp
    let addNewItem =
        let make caption = 
            Controls.Input "" 
            |> Validator.IsNotEmpty (caption + " should be set")
            |> Enhance.WithValidationIcon
            |> Enhance.WithTextLabel caption
        
        Formlet.Yield (fun name description -> name, description)
            <*> (make "Name")
            <*> (make "Description")
            |> Enhance.WithSubmitButton
            |> Enhance.WithLegend "Add new item"
            |> Formlet.Run (fun (name, description) ->
                    let item = { Id = 0; Name = name; Description = description}
                    // server call
                    let result = Remote<Handlers.Pagelets>.AddItem(item)
                    match result with
                    | Ok item -> 
                        items.Append(createItemRow item)
                        setStatus "Item was added successfully"  true
                    | Error message ->
                        setStatus message false
                ) 
```

```fsharp
type Pagelets(model : AppModel) = 
    [<Rpc>]
    member this.AddItem(item : Item) = 
        if model.Items |> Seq.exists (fun b -> b.Name = item.Name)
        then
            Error <| sprintf "Item with name '%s' already exists" item.Name
        else
            let newId = model.Items.Count
            let newItem = { item with Id = newId }
            model.Items.Add(newItem) 
            Ok <| newItem
```

Handler instances are provided by a factory – an implementation of `IRpcHandlerFactory`. This interface has only one method `Create : Type -> obj option`. The runtime invokes it to obtain the instance of a handler for the incoming request. The default implementation simply creates handlers on first requests using parameterless constructors and stores them in a cache so their behavior is similar to that of WebSharper 1.0 RPC methods. However, the user can substitute the default factory with a custom version that will use some IoC container for creating instances and managing their lifetime. It can be managed by calling the `IntelliFactory.WebSharper.RemotingPervasives.SetRpcHandlerFactory` function (you possibly have already noticed the similarity with the [`ControllerBuilder.SetControllerFactory`](http://msdn.microsoft.com/en-us/library/dd504976(VS.98).aspx) method). For our sample, we'll use [Autofac](http://code.google.com/p/autofac/) for creating both handlers and controllers.

```fsharp
type Application() = 
    inherit HttpApplication()
    
    let items = AppModel.CreateDummy()
    
    static member RegisterRoutes(routes : RouteCollection) = 
        routes.IgnoreRoute("{resource}.axd/{*pathInfo}")

        routes.MapRoute(
            "Default", // Route name
            "{controller}/{action}", // URL with parameters
            { controller = "Home"; action = "Index"} // Parameter defaults            
            )

    member this.Start() = 
        Application.RegisterRoutes(RouteTable.Routes) |> ignore

        let container = 
            let builder = new ContainerBuilder()
            builder.RegisterInstance(items) |> ignore // register model instance
            builder.RegisterType<Handlers.Pagelets>() |> ignore 
            builder.RegisterType<Handlers.Formlets>() |> ignore

            // register all controllers in current assembly
            builder
                .RegisterAssemblyTypes([| Assembly.GetExecutingAssembly() |])
                .AssignableTo<IController>() |> ignore
            builder.Build()

        let ctrlFactory = 
            { new DefaultControllerFactory() with
                override this.GetControllerInstance(requestContext, controllerType) =
                    if controllerType = null 
                    then 
                        let path = requestContext.HttpContext.Request.Path
                        let message = sprintf "controller for path %s not found" path 
                        raise <| new HttpException(404, message) 
                    else container.Resolve(controllerType : Type) :?> IController 
                override this.ReleaseController(controller) = () }
  
        ControllerBuilder.Current.SetControllerFactory(ctrlFactory)

        let handlerFactory = 
            { new IRpcHandlerFactory with
                member this.Create(handlerType) = 
                    container.Resolve(handlerType) |> Some }

        SetRpcHandlerFactory(handlerFactory)
```

The `Start` method is invoked from the `Application_Start` event handler. Inside `Start`, we perform the registration of routes, configure the Autofac container, and set up both the controller and handler factories.

![](/assets/164.png)

Full source code of this application is available as part of the standard WebSharper Sample Web Application (ASP.NET MVC) template for Visual Studio from the WebSharper 2.0 installer.
