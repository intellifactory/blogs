---
title: "Showcasing Feliz.Engine with WebSharper.React"
categories: "f#,websharper,feliz,proxy,fsadvent"
abstract: "A quick showcase of WebSharper proxying and Feliz.Engine by implementing a Feliz-like API for WebSharper.React."
identity: "-1,-1"
---

This article has been written for [F# Advent 2023](https://sergeytihon.com/2023/10/28/f-advent-calendar-in-english-2023/).

First, I'd like to thank [Sergey Tihon](https://twitter.com/sergey_tihon) for the opportunity to participate in this year's [F# Advent](https://sergeytihon.com/category/f-advent/)!

[Feliz](https://github.com/Zaid-Ajaj/Feliz) has been a helpful package for Fable developers in the last few years, for which I'd like to give huge props to [Zaid Ajaj](https://github.com/zaid-ajaj), allowing people to de-uglify their front-end code with getting rid of the `yourElement [] []` double-array known from virtually any framework, by giving you a DSL that lets you put everything into a simple "prop" collection and let the library itself figure out how to build your components from there.

And thanks to the work of [Alfonso Garcia-Caro](https://github.com/alfonsogarciacaro), there is now a framework-agnostic API called [Feliz.Engine](https://github.com/alfonsogarciacaro/Feliz.Engine), which'll be showcased here with a relatively simple "engine" for [WebSharper.React](https://github.com/dotnet-websharper/react).

To utilize this library, we first had to create a proxy for it, which is currently sitting inside a WIP repository, forked from the original.

--- 
### Wait, what's a proxy here?
Proxying - in this context - is the method of giving JS semantics to specific types or complete DLLs/packages to be able to use them in client-side WebSharper code. In this case, we had to go with the "easy route" of compiling the whole package with WebSharper, for which the steps were:
* Forking the `Feliz.Engine` repository
* Create a new WebSharper Proxy project, which code-wise is a regular F# project in our case
* If there were any missing members for BCL types, code a proxy for them, which looks something like:
```fsharp
open WebSharper

[<AutoOpen>]
module internal Prelude =
	[<Proxy(typeof<System.String>)>]
	type private StringProxy =
		[<Inline "$this.toLocaleUppercase()">]
		member this.ToUpperInvariant() = X<string>
		// here, X is the WebSharper equivalent of Fable's jsNative
```
* The "weird" step, add the origin project's source as a reference into the proxy project, so the `.fsproj` has lines like:
```xml
	<Compile Include="../Feliz.Engine/CssEngine.fs" />  
	<Compile Include="../Feliz.Engine/HtmlEngine.fs" />
	<None Include="wsconfig.json" />
```

We also include the WebSharper-generated wsconfig file, which, in our case, looks like this:
```json
{  
  "$schema": "https://websharper.com/wsconfig.schema.json",  
  "project": "proxy",  
  "proxyTargetName": "Feliz.Engine"  
}
```

After these steps, all we have to do is build a DLL, and use it in conjunction with the "original" package in other WebSharper projects.


---
### Today's topic
Thankfully, `Feliz.Engine` provides a really simple-to-use method through its generic Engine classes, such as `HtmlEngine` or `AttrEngine`, which are responsible for HTML nodes and HTML attributes,respectively. 

Note: `Feliz.Engine` is more of a "Feliz-like", framework-agnostic API, rather than a complete `Feliz` package, so the implementation of some features is up to library authors and contributors.

And to showcase this, I'll also take the opportunity to flaunt the ability to write `React` with `WebSharper`, by going through the steps of creating a simple/starter binding for `WebSharper.React`.

For this, we'll be using the latest available `WebSharper` package for this article, which, at the time of writing, is `7.0.3.364-beta3` for `WebSharper` and `WebSharper.FSharp`, and `7.0.2.364-beta3` for `WebSharper.React`.

---
### Implementing the React "Engine" 
First, we'll have to define our Node type that will handle all the different child, css, attribute and event handler props:

```fsharp
type Style = Style of string * obj
type Node =
	| Text of string
	| El of React.Element
	| Attr of string * obj
	| Styles of Style seq
	| Evt of string * obj
```
Since WebSharper.React already uses `string*obj` pairs for styling and attributes, we can just use these and separate attr/style/event with different union cases.

Also, since `Feliz.Engine`'s prop/styling keys must be converted to React-compatible keys, aka from `justify-content` to `justifyContent`, we'll also have to define a helper function to do this:
```fsharp
open System

let private toReactKey key =
	let arr = key.Split('-')
	
	let uppers =
		arr[1..arr.Length]
		|> Array.map (fun k ->
			let arr = k.ToCharArray()
			let firstUpper = arr[0].ToString().ToUpper()
			let rest = new String(arr[1..arr.Length])
			
			String.Concat(firstUpper, rest)
		)
		
	String.Join("", Array.append [|arr[0]|] uppers)

let inline private (|ToReactKey|) = toReactKey
```

Now that this is done, we can move onto the `Feliz` implementation.

For the easier part, we can implement our non-HTML engines first:
```fsharp
// the first param is for string-string pairs
// the second param is for string-bool pairs
let Attr = AttrEngine<Node>(fun a b -> Attr(a,b), fun a b -> Attr(a,b))

let Css = CssEngine<Style>(fun a b -> Style(a,b))

// we can also create and define other engines in the future, such as one for events
```
Here, AttrEngine requires a function which constructs one of its attributes that take string values, such as `aria-*="value"`, and a function which constructs attributes that take bool values, such as `disabled`.
And `CssEngine` is responsible for our styling properties, such as `display="flex"` through `engineInstance.displayFlex`.

For the `CssEngine<Style>` implementation, we'll also need a function which combines all the style props in a `Styles` case into one object, for that we can do the following:
```fsharp
open WebSharper

let private makeStyles (styles: Style seq) =
	let styleObj = JSObject()
	for (ToReactKey key,value) in styles do
		styleObj[key] <- value
	styleObj
```

To handle `Node` -> HTML transformation, we'll have to define the following:
* A function that returns an empty element, here we'll also define our helper for a `React.Fragment`:
```fsharp
open WebSharper.React
// helper for react fragments
let fragment nodes = 
	nodes
	|> Seq.choose (function
		| Text txt -> (Html.text >> Some) txt
		| El el -> Some el
		| _ -> None)
		|> ReactHelpers.Fragment
		|> El // back to an El node

let emptyEl () = fragment []
```

We'll also have to handle `Node.Element -> React.Element` conversion, here we'll only have to handle the `El` and `Text` cases:
```fsharp
	let toReact node = 
		match node with
		| El el -> el
		| Text txt -> Html.text txt
		| _ -> failwithf "Not a react element" // either this or an emptyEl()
```

* And a function that builds up a `React.Element` from a `Node seq`, which we'll also pass to the `HtmlEngine`'s "mk" parameter:

```fsharp
open WebSharper.React
open System.Collections.Generic

// to put JS.jsx calls through React.Element->Node transformations
let asNode = El

let makeEl name children =
	let rec addEls 
		(props: Dictionary<string,obj>) 
		(children: ResizeArray<_>)
		(nodes: Node seq) =
		let onElement = function
			| Text txt -> (Text txt) |> toReact
			| Elt elt -> elt
			|> children.Add
		nodes
		|> Seq.iter (fun node ->
			match node with
			| Text _ 
			| El _ -> onElement node
			| Styles styles -> props["style"] <- makeStyles styles)
			| Attr(ToReactKey key,value) 
			| Evt(ToReactKey key,value) -> props[key] <- value
	let props = Dictionary()
	let childArr = ResizeArray()
	addEls props childArr // note: there are definitely more functional ways
	let wsProps = 
		props
		|> Seq.map (fun kvp -> (toReactKey kvp.Key,kvp.Value))
	ReactHelpers.Elt name wsProps childArr
	|> El // mapping it back to a Node, toReact will handle rendering
```

* It also requires a `string -> 'Node` function, for which we can just pass the constructor of the `Node.Text` case.
```fsharp
type ReactHtmlEngine() =
	inherit HtmlEngine<Node>(makeEl, Text, emptyEl)
	with member _.evt = Evt // make the event bindings look a bit nicer

let html = ReactHtmlEngine()
```

We can also add a helper type to make it look and feel more like the "original" Feliz API:
```fsharp
let style = Css

type prop =
	static member styles = Styles
	static member children = fragment
	static member text = Text
```

Currently, we're binding `WebSharper.React.Html.on.*` events to Nodes with either a `html.evt` or an `Evt` call, so that is a bit verbose, which could be solved in the future with something like an `EventEngine` implementation, which might be handled in a future blog post.

But for a shortcut, we can just export a method that constructs a `Node.Evt` case from an event handler, and use module aliases for a shorter syntax, such as:
```fsharp
// either this or putting it inside the "prop" type's definition
type prop with
	static member evt = Evt

// in frontend/DSL code
module on = WebSharper.React.Html.on
```

Now, we can write our react code (showcased by the good ol' reliable counter example) in a somewhat friendlier way, such as:

```fsharp
namespace Sample

open Feliz
open WebSharper
open WebSharper.JavaScript
open WebSharper.React
open WebSharper.Feliz.Engine.React

[<JavaScript>]
module Client =
	let CounterFunctionExample() =
		React.CreateElement((fun _ ->
		
			let count, setCount = React.UseState 0
			
			html.div [
				prop.children [
					html.button [
						prop.text "Increment"
						html.evt (Html.on.click (fun _ -> setCount.Invoke(count+1)))
					]
					html.span [
						prop.text $"{count}"
					]
					html.button [
						prop.text "Decrement"
						html.evt (Html.on.click (fun _ -> setCount.Invoke(count-1)))
					]
				]
			
		] |> toReact), ())


	[<SPAEntryPoint>]

	let Main () =
		let root = ReactDOM.ReactDomClient.CreateRoot (JS.Document.GetElementById("root"))
		
		html.div [
			prop.children [
				html.div [
					prop.styles [
						style.displayFlex
						style.flexDirectionColumn
						style.justifyContentSpaceBetween
						style.backgroundColor color.aliceBlue
					]
					prop.children [
						JS.jsx """<h1>Hello there!</h1>""" |> asNode
						CounterFunctionExample() |> asNode
					]
				]
			]
		]
		|> toReact
		|> root.Render
	
```

As always, we can extend this implementation to fit our needs with more `Node` cases/extra kinds of engines, or write new implementations for specific UI frameworks, such as FluentUI, Bulma or WebSharper.UI. 

As an added bonus, you can also use this new DSL in Elmish projects!

If you want to play around with this [Feliz.Engine](https://github.com/alfonsogarciacaro/Feliz.Engine) fork and the demo project shown above, it's available on [this personal GitHub repository](https://github.com/GyGerg/WebSharper.Feliz.Engine). You can find the proxy project in `src/WebSharper.Feliz.Engine`, and the sample project in `samples/WebSharper.Feliz.Engine.React`, with a small "how-to-run" readme.

### Happy Holidays!