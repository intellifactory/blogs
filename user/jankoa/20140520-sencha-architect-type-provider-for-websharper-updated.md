---
title: "Sencha Architect type provider for WebSharper updated"
categories: "senchaarchitect,extjs,senchatouch,f#,websharper"
abstract: "This release makes it even more convenient to build Ext JS and Sencha Touch applications with WebSharper."
identity: "3866,77200"
---
This release makes it even more convenient to build Ext JS and Sencha Touch applications with WebSharper.

See the updated [documentation](http://websharper.com/extensions/senchaarchitect) and [sample](http://websharper.com/samples/SenchaArchitect) pages.

Changes to the Sencha Architect type provider:

 * Tools on a panel with an `itemId` have provided getter properties.
 * Optional `Inherit = true` argument on the type provider. This makes the wrapper types inherit their component class. Switching to this makes the `.self` property unnecessary in most of the cases, but you can still use it for upcast. On the downside, generated getter properties for items are harder to find in IntelliSense as they are mixed with the members of the component class. This has no effect on generated JavaScript code.
 * Parser fixes.

Recent changes to the Ext JS/Sencha Touch bindings:

 * Breaking change: Function arguments that are not documented in the Sencha docs are now typed `EcmaScript.Function` in the binding. Adding an `As` cast may be required to older code.
 * All config objects have an inlined `.With` method defined with two overloads. It can add a single key-value pair or copy all the fields from another config object.

<a href="http://websharper.com/samples/SenchaArchitect"><img src="http://i58.tinypic.com/11uburt.png"></a>
