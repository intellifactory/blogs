---
title: "WebSharper 3.0.8-alpha published"
categories: "f#,websharper"
abstract: "This release of WebSharper streamlines a lot of the namespaces, making it more intuitive to use than ever. It is a breaking update, and this article describes all the (mostly trivial) changes that need to be done to existing projects."
identity: "4155,77618"
---
As we advance closer to the stable release of WebSharper 3.0, more and more of the breaking changes we have wanted to implement for a long time are coming out. The main theme of the just-released version 3.0.8-alpha is naming conventions streamlining. With these changes, WebSharper should be more intuitive to use than ever. The changes necessary to update an existing project are fairly simple and detailed in the comprehensive guide below.

## Naming changes

In names below, `IF` is short for `IntelliFactory` and `IFWS` for `IntelliFactory.WebSharper`.

### JavaScript standard library

Until now, the JavaScript standard library was split into a number of namespaces and modules, making it sometimes difficult to find where something was. The new namespace `IFWS.JavaScript` is now the home of everything built in the browser, whether it's basic language constructs, DOM, HTML5, etc.

The general rule is:
**If a module contains client-side code, you should `open IntelliFactory.WebSharper.JavaScript`.
If it only contains server-side code, you should not.**

For the purpose of the above rule, WIG definitions ("Extension" projects) are not considered client-side code, since they are run *during compilation* to generate inlines. In fact, it is important *not* to `open IFWS.JavaScript` in a WIG definition because of a conflict on the `(?)` operator.

In more detail:

 * `IFWS.EcmaScript` and `IFWS.Html5` are removed and all types defined under them are now under `IFWS.JavaScript`. The module `IFWS.EcmaScript.Global` is merged with the old `IFWS.JavaScript` and renamed to `IFWS.JavaScript.JS`. A few HTML5 items have also been cleaned up, and more will certainly come in the future. For example, code that used to read:

    ```fsharp
    open IntelliFactory.WebSharper

    let a = EcmaScript.Date.Now()
    let b = new Html5.Float32Array()
    let c = Global.NaN
    let d = JavaScript.Undefined
    let e : EcmaScript.WebGLRenderingContext = (* ... *)
    let f : Html5.Position = (* ... *)
    JavaScript.Log "Some text on the console"
    ```

    now reads:

    ```fsharp
    open IntelliFactory.WebSharper
    open IntelliFactory.WebSharper.JavaScript

    let a = Date.Now()
    let b = new Float32Array()
    let c = JS.NaN
    let d = JS.Undefined
    let e : WebGL.RenderingContext = (* ... *)
    let f : Geolocation.Position = (* ... *)
    Console.Log "Some text on the console"
    ```

    `JS` now basically contains all top-level values and language constructs defined in JavaScript. This includes `JS.Window` and `JS.Document`, which obsolete `Html5.Window.Self` and `Dom.Document.Current`, respectively.

 * `IFWS.Dom` is renamed to `IFWS.JavaScript.Dom`.

 * The auto-opened `IFWS.Pervasives` is split in two:

    * `IFWS.Pervasives` contains definitions useful for both client-side and server-side code, such as attributes (`[<JavaScript>]`, `[<Inline>]`, `[<Rpc>]`, etc).

    * `IFWS.JavaScript.Pervasives` contains client-side definitions, such as:

        ```fsharp
        open IntelliFactory.WebSharper.JavaScript

        // notation for plain JavaScript object creation
        let g = New ["x" => 12; "y" => 14]

        // plain JavaScript field access
        let h = e?x

        // "dotted" operators, inlined to the corresponding JavaScript operator
        if g ===. h then ()
        ```

### HTML combinators

 * The namespaces `IF.Html` and `IFWS.Html` have been a constant source of confusion for beginners. In WebSharper 3.0.8, they have been renamed to `IFWS.Html.Server` and `IFWS.Html.Client`, respectively. Hopefully this should make it more obvious when one or the other is needed.

 * Tag and attribute combinators that were previously under a `HTML5` module are now grouped back together with the "normal" ones. We judged that it doesn't make sense to separate them out anymore.

 * The type `IF.Html.Element<'T>` was originally intended to allow embedding various types of client-side controls. But ultimately, only `Web.Control` has ever been used for this purpose. Therefore, we made the corresponding new `IFWS.Html.Server.Element` and all associated types (such as `INode` and `Attribute`) non-parametric.


### Formlets

The following namespaces have been renamed for consistency with Sitelets and Piglets:

 * `IF.Formlet --> IF.Formlets`
 * `IFWS.Formlet --> IFWS.Formlets`
 * `IFWS.Formlet.JQueryUI --> IFWS.Formlets.JQueryUI` (extension)
 * `IFWS.Formlet.TinyMce --> IFWS.Formlets.TinyMce` (extension)

The NuGet package WebSharper.Formlet.JQueryUI has also been renamed to WebSharper.Formlets.JQueryUI.


## Functionality changes

### Web.Control and HTML type hierarchy

In preparation for an upcoming tighter integration of UI.Next, we changed the behavior of `Web.Control`. Namely, the `Body` property doesn't have the type `IPagelet` anymore, but a new interface `IControlBody` that is implemented by pagelets and will be soon implemented by UI.Next `Doc`s. As a consequence:

 * The interface `IPagelet` is replaced by an abstract class `Pagelet`, which implements `IControlBody` appropriately.

 * The `Body` property of `Html.Client.Element`s now has type `Dom.Node` instead of `Dom.Element`. Some methods (such as `JQuery.Append`) require a `Dom.Element`; in this case, replace `.Body` with `.Dom`:

    ```fsharp
    // old code
    open IntelliFactory.WebSharper.Html
    open IntelliFactory.WebSharper.JQuery

    let e = Div [Text "some content"]
    JQuery.Of("#myElt").Append(e.Body).Ignore

    // new code
    open IntelliFactory.WebSharper.Html.Client
    open IntelliFactory.WebSharper.JQuery

    let e = Div [Text "some content"]
    JQuery.Of("#myElt").Append(e.Dom).Ignore
    ```

### Assembly referencing

This one gets a bit more technical and is related to the WebSharper build process. WebSharper contains a custom MSBuild task written in F#, which is in charge of compiling code to JavaScript and extracting the compiled JavaScript code into a web project, among other things. Until now, this task was also in charge of adding all the WebSharper assembly references to the project just before running the F# compiler. This had several unfortunate consequences: the solution explorer in Visual Studio did not show the WebSharper references, and in some cases on Mono (such as when compiling a full solution in CloudSharper), XBuild would fail to add the references correctly. In this new release, we switched to a more conventional scheme where references are added to the project file itself by `nuget install`. This more standard behavior should harbor fewer opportunities for problems.

## Conclusion

Phew, that was a long one! Overall these changes are fairly simple to apply to an existing project, and we are confident that they remove a number of idiosyncrasies that hinder the learning of WebSharper for beginners.

If you encounter any problems porting a project to the latest WebSharper, don't hesitate to tell us on [the issue tracker](https://github.com/intellifactory/websharper/issues). Happy coding!
