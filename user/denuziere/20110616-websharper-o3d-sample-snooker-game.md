---
title: "WebSharper / O3D sample: Snooker game"
categories: "o3d,webgl,f#,websharper"
abstract: "We translated the snooker game sample from the official O3D website to WebSharper. The result is on par with the original in terms of performance, with 25% shorter source code and extra code safety."
identity: "2059,74891"
---
We are happy to present to you <a href=http://www.websharper.com/samples/O3D>the first game</a> coded using WebSharper. It is a translation from one of O3D's biggest samples: the snooker.

<img src="http://intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=195">

This WebSharper snooker is on par with <a href=http://o3d.googlecode.com/svn/trunk/samples_webgl/o3d-webgl-samples/pool.html>its JavaScript counterpart</a> in terms of game speed, showing the performance of our generated code. But the source code is actually about 25% shorter than the original. Moreover, the extra safety brought by F# and WebSharper would have made its development easier and faster.

## Why is it safer?

When we developed the WebSharper bindings to O3D, we brought a few modifications which added some extra safety.

First, mathematical operations on vectors, matrices and quaternions are more strongly typed. Indeed, while the JavaScript API uses arrays to represent these data types, we use tuples. Therefore the number of elements is always known. For example, if you try to compute the dot product of two vector of different sizes (using `o3djs.math.dot`) in JavaScript, it will silently return an erroneous value. However, in WebSharper, the error is detected at compile time by the type system, because there is no overload on `O3DJS.Math.Dot` taking two tuples of different sizes.

Another element that makes O3D in JavaScript unsafe is its use of methods whose return type depends on the arguments. An example is `Pack.createObject`. It takes a string designating a type, and returns an object of the given type. In WebSharper, there is an appropriately typed version of this method for each possible return type. So instead of:

```fsharp
var material = pack.createObject('o3d.Material');
```
You would write:

```fsharp
let material = pack.CreateMaterial()
```

This makes its use a lot safer, and eliminates the risk for hard-to-find bugs due to typos in the string.

## Why is it shorter?

When we translated the snooker to WebSharper, we modified it to a more idiomatic functional style. For example, a loop like the following in JavaScript:

```js
for (i = 0; i < arr.length; ++i)
{
  var element = arr[i];
  // do things with element
}
```

is written in F# as:

```fsharp
arr |> Array.iter (fun element ->
  // do things with element
)
```

This style makes it more immediately apparent that we are actually iterating over the elements of arr, and that no other criterion interferes with the control flow. As an appreciable extra, it makes the code shorter by several lines.

<img src="http://intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=196">

So, go ahead, <a href=http://www.websharper.com/samples/O3DRun>try it</a> and don't hesitate to have a look at <a href=http://www.websharper.com/samples/O3D>the source</a>!
