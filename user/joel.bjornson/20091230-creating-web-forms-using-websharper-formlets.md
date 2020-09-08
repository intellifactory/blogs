---
title: "Creating web forms using WebSharper Formlets"
categories: "f#,websharper,formlets"
abstract: "In this post I give a short introduction to the WebSharper formlet library."
identity: "1913,74745"
---
One of the most common (and tedious) tasks in web development is creating web forms for collecting user information. It requires functionality for:

 * Looking up form data and converting it into server-side data types.
 * Validating the data, i.e. ensure that the entered values are correct and meaningful.
 * Create feedback to the user if the submitted form contains invalid or incomplete data.

The WebSharper framework provides a neat solution to tackle these problems based on a concept called [Formlets](https://web.archive.org/web/20110520052730mp_/http://groups.inf.ed.ac.uk/links/formlets/). A formlet is an abstraction over web forms allowing you to specify forms in a declarative style without having to worry about the implementation details. Some of its merits include:

 * Type safety - Web forms of any type can specified.
 * Composabilty - Complex formlets are created by composing simpler ones.
 * Automatic error handling - Notifying the user of missing or incorrect field values etc.

Consider the following example for creating a formlet for reading a name and an email address:

```fsharp
[<JavaScriptType>]
type Person = { Name : string; Email: string }

    [<JavaScript>]
    let PersonFormlet : Formlet<Person> =
        let nameF =
            Controls.Input "" 
            |> Validator.IsNotEmpty "Empty name not allowed" 
            |> Controls.Enhance.WithLabel "Name"
        let emailF =
            Controls.Input "" 
            |> Validator.IsEmail "Please enter valid email address"
            |> Controls.Enhance.WithLabel "Email"
        Formlet.Yield (fun name email -> { Name=name; Email=email })
        <*> nameF
        <*> emailF
```

The formlet may be rendered as:

> **Review note** <br />
> This resource needs to be re-added.
> ![](//www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=124)

Note that:

 * The `PersonFormlet` definition creates a formlet parametrized with the custom type `Person`.
 * Since it's a formlet it can be rendered as a web form on its own or be composed with other formlets.
 * It contains validation logic to guarantee that the name field is not empty and the email address is of correct format.

In other words, only the very necessary information of which controls to add, what validation logic to apply and how to aggregate the result of the sub-forms is required. All the hard work of creating HTML elements, extracting the form data and outputting error messages is automated.

The way you compose formlets is based on a concept called *Applicative Functors* or *Idioms*. The *Idiom* concept provides a nice way of composing formlets but is not powerful enough to express dependencies between different formlets.

So, in addition to support for composition using *Idioms*, WebSharper formlets comes with a convenient way of expressing dependent formlets using F# computation expressions (or Monads).
However, you do not need to grasp these concepts in order to use the Formlet library. Looking at a few examples should be enough to get the idea of how to use them.

The following example illustrates how to express dependent formlets:

```fsharp
[<JavaScriptType>]
type ContactData = | Phone of string | Email of string                    

[<JavaScript>]
let ContactForm : Formlet<ContactData> =
    let contactTypeF = 
        Controls.Select [("Phone", "P"); ("Email", "E")] (Some "P")
    Formlet.Do {
        let! contactType = contactTypeF
        let! contact =
            match contactType with
            | "P" -> 
                Input "" 
                |> Enhance.WithLabel "Phone" 
                |> Formlet.Map Phone
            | _ ->
                Input "" 
                |> Enhance.WithLabel "Email" 
                |> Validator.IsEmail "Please enter valid email address"
                |> Formlet.Map Email
        return contact                        
    }
```

Here a formlet for reading contact information is created. What's interesting is the different shape of the contact information (either a phone number or an email address).

The contact formlet therefore contains a select box for selecting the contact type. The selected value is then used to trigger the rendering of either an email form or a phone number form. Each time the user changes the selected option, the dependent formlet is updated.

> **Review note** <br />
> This resource needs to be re-added.
> ![](//www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=122)

Once again, the definitions are given in a declarative way. All the implementation logic for how to dynamically update the DOM nodes etc may be conveniently swept under the carpet.

The WebSharper formlet library is designed with the intention of making web form creation as painless as possible.

To see some running examples of formlets have a look at the demo section[^1].

[^1]: This link is dead and has been removed. You can find formlet examples on [Try WebSharper](https://try.websharper.com), filtering for snippets that use `UI.Formlets`. These formlets are an enhanced, reactive version of the original formlet library. You can find more information the [WebSharper.Forms README](https://github.com/dotnet-websharper/forms).
