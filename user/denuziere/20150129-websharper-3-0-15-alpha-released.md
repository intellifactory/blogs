---
title: "WebSharper 3.0.15-alpha released"
categories: "javascript,web,f#,websharper"
abstract: ""
identity: "4205,77675"
---
Here comes another release of WebSharper 3.0 alpha! This release focuses on small fixes and improvements, as well as a couple new proxies for useful classes such as `System.Random` and `System.Guid`.

## Changes

### Improvements

 * WIG: The `(?)` operator for naming a method argument now properly overrides the JavaScript field access operator. This means that you can now `open IntelliFactory.WebSharper.JavaScript` in a WIG project, provided that you do it **before** `open IntelliFactory.WebSharper.InterfaceGenerator`.
 * Added xml documentation for attributes and sitelets.
 * Added `Sitelets.UserSession.LoginPersistent` to create login sessions that persist beyond the current browser session.
 * Added client-side `Remoting.UseHttps()` which forces all RPC calls to use HTTPS.


### New proxies

The following classes and methods can now be used from the client side.

 * `System.Random`
 * `System.Guid`
 * `System.String.TrimStart/End`


### Fixes

 * Fix [#315](https://github.com/intellifactory/websharper/issues/315): `Sitelet.Protect` login redirect now uses `RedirectTemporary` instead of `Redirect`.
 * Fix [#322](https://github.com/intellifactory/websharper/issues/322): Compile-time error when a project's `Name` and `AssemblyName` are different.
 * Fix [#323](https://github.com/intellifactory/websharper/issues/323): Fields for primary constructor arguments shadow inherited ones
 * Fix [#325](https://github.com/intellifactory/websharper/issues/325): Add `Console.Log` overload with a single argument


## Future plans

We have two final major changes to push for WebSharper 3.0, which affect the typing conventions for tupled and multiple argument functions ([see the work in progress](https://github.com/intellifactory/websharper/commits/no-rt-tuple)) and the interface generator API. Our current plan is to have another alpha release for this (and other, smaller changes), and ideally to release the first stable version of WebSharper 3.0 before the end of February.

Happy coding!
