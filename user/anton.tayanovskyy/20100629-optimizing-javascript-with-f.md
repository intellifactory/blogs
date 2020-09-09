---
title: "Optimizing JavaScript with F#"
categories: "f#,websharper"
abstract: "These days WebSharper trenches are teeming with activity as we are busy preparing the next major release of the platform. One of the F#-to-JavaScript compiler highlights of the new release is a host of new optimizations on the JavaScript output. Designing these optimizations with F# was quite rewarding, and I am sharing some of the eurekas in this blog."
identity: "1449,74659"
---
These days WebSharper trenches are teeming with activity as we are busy preparing the **next major release** of the platform. One of the F#-to-JavaScript compiler highlights of the new release is a host of optimizations on the JavaScript output. Designing these optimizations with F# was quite rewarding, and I am sharing some of the eurekas in this blog.

The need for optimizing JavaScript is very peculiar. This is not about delivering top wall-clock performance, but rather delivering the most compact (in Release mode) and most readable (in Debug mode) source. In the case of WebSharper, it is also about relieving the compiler and the macro writer from the burden of emitting optimal code. Essentially, we would like to conform to the Macro Writer's Bill of Rights[^1], for JavaScript.

The optimizations covered today include:

 * constant folding
 * inlining
 * redex elimination
 * common subexpression elimination

The problem with optimizing JavaScript is that it is *complex*. There are expressions, statements, objects, field assignments and what not. Scoping of variables is not functional, there are no let statements (we target the 3rd edition of ECMA-262). Any of the above optimizations are non-trivial on the complete JavaScript AST.

For this and other reasons, we ended up designing an intermediate representation, Core JavaScript. Unlike complete JavaScript, Core JavaScript is functional, simple, and very easy to optimize. In fact, Core JavaScript expressions instantiate Core Scheme expressions parameterized by a primitive set. Here is the definition of a Core Scheme expression:

```fsharp
type Expression<'P> =
    | Apply of E<'P> * list<E<'P>>
    | Assign of Id * E<'P>
    | If of E<'P> * E<'P> * E<'P>
    | Lambda of list<Id> * E<'P>
    | Let of Id * E<'P> * E<'P>
    | LetRecursive of list<Id * E<'P>> * E<'P>
    | Primitive of 'P * list<E<'P>>
    | Undefined
    | Var of Id
```

The good news is that while Core JavaScript is fully as expressive and in fact very close to the complete JavaScript, we can now write optimizations as transformations on Core Scheme expressions that preserve the (simple) Scheme semantics. For example, common subexpression elimination looks for matching pure, ground subexpressions, and lifts them to a `Let` binding. And of course not having to deal with expression/statement distinction, `this` argument passing convention, and the like is the joy for the compiler or macro writer.

**Example 1**: Eliminating Common Subexpressions

```fsharp
let x = Id "x"
let y = Id "y"
let this = Id "this"
E.Array [
    E.Function this [x; y] (!^this .% !^x ..% [!^y])
    E.Function this [x; y] (!^this .% !^x ..% [!^y])
]
```

Note: `( !^ ) :: Id -> Expression` lifts identifiers to expressions.

This outputs:

```javascript
(function ()
 {
   var _;
   function _(x, y)
   {
     return this[x](y);
   }
   return [_, _];
 })()
```

**Example 2**: Inlining and Constant Folding

```fsharp
E.Let y (E.Int 1) (
    E.Let x (E.Int 2) (
        !^x +% !^y
    )
)
```

Outputs:

```javascript
(function ()
 {
   return 3
 })()
```

**Example 3**: Redex Elimination and Constant Folding

```fsharp
E.Function this [x] (!^x +% E.String "!") 
    ..% [E.String "Hello"]
```

Outputs:

```javascript
(function ()
 {
   return "Hello!"
 })()
```

Writing the optimizations was a joy. For example, the inlining transformation is within 20 lines of F#.

The largest contribution to the joy is the fact that the underlying engine takes care of JavaScript insanity.

For instance, using the same *suggested* variable names gives valid JavaScript code by disambiguating variables:

```fsharp
E.Let x (L?foo ..% []) (
   E.Let x (L?bar ..% [!^x]) (
      L?baz ..% [!^x]
   )
)
```

Outputs:

```javascript
(function ()
 {
   var x, x1;
   x = foo();
   x1 = bar(x);
   return baz(x1);
 })()
```

As a final example, `this`-style variables are automatically handled when captured, for example:

```fsharp
E.Function this1 [] (
    E.Function this2 [] (
        !^this1 +% !^this2
    )
)
```

Outputs:

```javascript
(function ()
 {
   return (function ()
           {
             var _this = this;
             return (function ()
                     {
                       return _this + this;
                     });
           });
 })()
```

How will these improvements benefit the WebSharper ecosystem?

 * the output code of WebSharper projects will be leaner
 * macros will be a lot easier to write
 * the compiler will be smaller and simpler

In subsequent posts on the JavaScript optimization effort I will cover:

 * JavaScript Code Model (a form of CodeDOM for JavaScript)
 * Global and local tail call optimizations

Stay tuned!


[^1]: This link is no longer available.
