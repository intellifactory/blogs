---
title: "Announcing Formlets for jQuery Mobile"
categories: "f#,websharper,formlets,jquery,jquerymobile"
abstract: "The WebSharper Extensions for Formlets for jQuery Mobile give you the power of jQuery Mobile combined with the succinctness of formlets."
identity: "1903,74735"
---
The WebSharper Extensions for Formlets for jQuery Mobile give you the power of jQuery Mobile combined with the succinctness of formlets. It can be downloaded from the extension page[^1]. You also need the latest jQuery Mobile extension installed.

Basically, everything from this extension is compatible with the native WebSharper formlets, so it also uses the same syntax:

```fsharp
let theme = Theme.C
Formlet.Yield (fun a b -> a, b)
<*> (Controls.TextField "" theme
    |> Enhance.WithTextLabel "Username:")
<*> (Controls.Password "" theme
    |> Enhance.WithTextLabel "Password:")
|> Enhance.WithSubmitButton "Log in" theme
|> Formlet.MapResult (function
    | Success (_, pass) when pass.Length < 8
        -> Failure [ "Wrong credentials." ]
    | n -> n)
|> WithJQueryMobileLayout theme
```

![](//i.imgur.com/to90A.png)

As all first-class formlets, you can use jQuery Mobile controls in a dependent formlet:

```fsharp
let theme = Theme.D
Formlet.Do {
    let! a, b =
        Formlet.Yield (fun a b -> a, b)
        <*> Controls.Slider -25. 25. 5. theme
        <*> Controls.Slider -25. 25. 5. theme
    do! Formlet.OfElement (fun _ ->
        P [
            Text (
                if a = 0. then
                    "Any x solved the equation " + string a +
                        " * x + " + string b + " = 0!"
                else
                    let solution = -1. * b / a
                    string a + " * " + string solution + " + " +
                        string b + " = 0"
            )
        ])
    return ()
}
```

![](//i.imgur.com/R2Bgw.png)

And finally, a larger example showing more controls:

```fsharp
let msg = Span []

let link link =
    Controls.CustomButton
        {
            Inline = true
            Icon = None
            Theme = theme
            Link = "#sample-" + link
            Transition = Transition.Fade
            Text =
                link.Substring(0, 1).ToUpper() +
                    link.Substring 1
        }

Div [
    [
        link "buttons"
        link "inputs"
        link "basic"
    ]
    |> GroupControls true
    :> IPagelet

    msg :> _

    Br [] :> _

    Formlet.Do {
        let! _ = Controls.TextField "Type me!" theme
        let! _ =
            Controls.FlipToggle "ON" "OFF" true theme
            |> Enhance.WithTextLabel "This Form:"
            |> Formlet.MapResult (function
                | Success false -> Failure []
                | n -> n)
        let animals =
            [
                "Fish", 1.
                "Dog", 2.
                "Cat", 3.
            ]
        let! an1, [an2], [ b; i; u ] =
            Formlet.Yield (fun a b c -> a, b, c)
            <*> (Controls.RadioGroup animals 2. false theme
                |> Enhance.WithTextLabel "Animal one:")
            <*> (Controls.SelectMenu
                    { SelectConfiguration<_>.Basic animals 2. with
                        NativeSelect = false } theme
                |> Enhance.WithTextLabel "Animal two:")
            <*> (Controls.CheckboxGroup [ "b", true; "i", true; "u", false ] true theme
                |> Enhance.WithTextLabel "Format:"
                |> Formlet.Map List.ofArray)

        msg.SetCss("font-weight",
            if b then
                "bold"
            else
                "normal")

        msg.SetCss("font-style",
            if i then
                "italic"
            else
                "normal")

        msg.SetCss("text-decoration",
            if u then
                "underline"
            else
                "none")

        let valOfAn = ( * ) 10.
        let! v =
            Controls.Slider 0. 100. (valOfAn an1 + valOfAn an2) theme
            |> Enhance.WithTextLabel "Value:"
        msg.Text <- string v
        return! Formlet.Never()
    }
    |> WithJQueryMobileLayout theme
    :> _
]
```

![](http://imgur.com/lhPBG.png)


[^1]: Link is dead and was removed. Check the [NuGet.org/WebSharper](https://www.nuget.org/packages?q=websharper) page for a list of public WebSharper extensions.
