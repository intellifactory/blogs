---
title: "WebSharper 2.5.127 released"
categories: "f#,websharper"
abstract: "This version brings small fixes and RPC improvements"
identity: "4056,77462"
---
## Additions to RPC functionality

`Map` and `Set` types are now usable in `RPC` functions.

If you are returning these to client side, be sure that the type of the set elements or map keys have the same comparison logic the server and client side, or else it can lead to unexpected behavior as the returned collection is not re-sorted. However, you don't have to worry about this if you don't explicitly overload the CompareTo method with a JavaSript `Inline` working differently than the method body running in .NET.

## Bugfixes

 * A compile error is given if a non JSON-encodable type is used in an RPC signature.
 * `Date.now` and the `Async` scheduler now works on IE8.
 * No more deprecated warning about using `nodeValue`.
 * In WIG, `T<SomeGenericType<_>>.[YourTypeDefinition]` now translates properly to indicate the type `SomeGenericType<YourType>`.
