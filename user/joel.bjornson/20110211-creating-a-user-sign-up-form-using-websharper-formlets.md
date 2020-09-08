---
title: "Creating a user sign-up form using WebSharper Formlets"
categories: "f#,websharper,formlets"
abstract: "In this post I'll highlight some of the features of WebSharper formlets by implementing a user sign-up form."
identity: "1907,74739"
---
In this post I'll highlight some of the features of WebSharper formlets by implementing a user sign-up form. The examples make use of some common patterns including:

 * Adding validation.
 * Using dependent formlets.
 * Submitting form results to the server.

For a general introduction to WebSharper formlets see: [Working with Formlets](https://github.com/dotnet-websharper/forms)[^1].

Let's start simple by constructing a basic formlet for collecting a user name and a password. The formlet below consists of two text-box fields for entering name and password, along with submit and reset buttons. The text-box fields are decorated with labels. The formlet produces values of the custom type `User`:

```fsharp
type User =
    {
        Name : string
        Password : string
    }
```

```fsharp
[<JavaScript>]
let SignUpForm =
    let userName =
        Controls.Input ""
        |> Enhance.WithTextLabel "User Name"
    let password =
        Controls.Password ""
        |> Enhance.WithTextLabel "Password"
    let user =
        Formlet.Yield (fun name password -> 
            {Name = name; Password = password}
        )
        <*> userName
        <*> password
    user
    |> Enhance.WithSubmitAndResetButtons
    |> Enhance.WithFormContainer
```

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=169)

The text-box formlets are composed using `Formlet.Yield` and `<*>`. This way of grouping formlets is called *static composition*, since the constituting formlets do not depend on each other.

Next, we consider some enhancements. For example, adding validation in order to restrict the user from submitting and empty name and require a minimum length of the password. We could also enforce the user to retype the password.

Here is an extended version implementing both the validation and password retyping features.

```fsharp
[<JavaScript>]
let SignUpForm =
    let userName =
        Controls.Input ""
        |> Validator.IsNotEmpty "Enter name"
        |> Enhance.WithValidationIcon
        |> Enhance.WithTextLabel "User Name"
    let password =
        Formlet.Do {
            let! pw = 
                Controls.Password ""
                |> Validator.Is (fun s -> s.Length >= 5) 
                    "Minimum required length of password is five characters."
                |> Enhance.WithValidationIcon
                |> Enhance.WithTextLabel "Select a Password"
            return!
                Controls.Password ""
                |> Validator.IsEqual pw "Passwords must match."
                |> Enhance.WithValidationIcon
                |> Enhance.WithTextLabel "Re-type password"
        }
    let user =
        Formlet.Yield (fun name password -> 
            {Name = name; Password = password}
        )
        <*> userName
        <*> password
    user
    |> Enhance.WithSubmitAndResetButtons
    |> Enhance.WithFormContainer
```

To implement the password retyping functionality, dependent formlets are used. In contrast to static composition, this is called *dynamic composition*. The password formlet in the code above is defined using the formlet builder (`Formlet.Do {..}`). The first component is a text-box formlet producing a string value. Everytime this formlet triggers a value a new text-box formlet is added. This formlet is enhanced with validation by comparing the value with the one produced by the first text-box. When the passwords do not match, a red icon is displayed and the the submission of the form is deactivated.

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=170)

The final step is to submit the form data by calling a server side method. Let's assume the following function:

```fsharp
module Server =
    [<Rpc>]
    let SignUpUser(user: User) = ..
```

This function would contain the code for registering the uer and return a boolean value indicating whether succeeding or not.

To implement the submission, again dependent formlets can be utilized. In the final version below, the data from the user form is collected and sent to the server. Depending on the result of the server response one of two different messages are displayed to the user.

Here is the complete code:

```fsharp
[<JavaScript>]
let SignUpForm =
    let userName =
        Controls.Input ""
        |> Validator.IsNotEmpty "Enter name"
        |> Enhance.WithValidationIcon
        |> Enhance.WithTextLabel "User Name"
    let password =
        Formlet.Do {
            let! pw = 
                Controls.Password ""
                |> Validator.Is (fun s -> s.Length >= 5) 
                    "Minimum required length of password is five characters."
                |> Enhance.WithValidationIcon
                |> Enhance.WithTextLabel "Select a Password"
            return!
                Controls.Password ""
                |> Validator.IsEqual pw "Passwords must match."
                |> Enhance.WithValidationIcon
                |> Enhance.WithTextLabel "Retype password"
        }
    let user =
        Formlet.Yield (fun name password -> 
            {Name = name; Password = password}
        )
        <*> userName
        <*> password

    Formlet.Do {
        let! user = Enhance.WithSubmitAndResetButtons user
        return! 
            Formlet.OfElement <| fun _ ->
                if Server.SignUpUser user then
                    Div [Attr.Class "infoPanel"] -< [
                        Text "Thanks, you are are now signed up."
                    ]
                else
                    Div [Attr.Class "warningPanel"] -< [
                        Text "Sorry, signing up failed. 
                                        Please select a different user name."
                    ]
    }
    |> Enhance.WithFormContainer
```

And the corresponding form rendered in a browser:

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=171)

> **Review note** <br />
> This resource needs to be re-added.
> ![](http://www.intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=172)


[^1]: This link has been updated to point to WebSharper reactive forms, which include further pointers on formlets.
