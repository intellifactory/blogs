---
title: "Building Mobile App with F# and WebSharper"
categories: "f#,websharper,ui,android,capacitor,vite,fsadvent"
abstract: ""
identity: "-1,-1"
---

Hello, everyone! In this article, we will explore how to create mobile applications in F# with WebSharper. Whether you're new to mobile development or functional programming, this guide will help you get started with practical steps and examples.

## Introduction

| Before adding a image | After adding a image and drawing |
|--|--|
| [![](/assets/picdrawapp-before-adding-image.jpg)](/assets/picdrawapp-before-adding-image.jpg) | [![](/assets/picdrawapp-after-adding-image.jpg)](/assets/picdrawapp-after-adding-image.jpg) |


This mobile app is a simple Android application that allows users to take or choose a picture, annotate it by drawing directly on the image, and then save or share it. This guide will show you how to develop this app using F# and WebSharper. Whether you’re interested in functional programming or mobile development, this project provides a hands-on experience.

While the focus is on Android, you can create an iOS app from the same codebase. To build for iOS, you'll need access to a MacBook, install Xcode, and configure the iOS platform using Capacitor. Refer to the Capacitor documentation for detailed steps on setting up an iOS build.

## Why F# and WebSharper?

F# is a functional-first programming language that emphasizes immutability and concise syntax, making it ideal for modern web and mobile applications. WebSharper enables you to create interactive web and mobile apps with minimal hassle.

This article guides you through building a mobile application we call 'PicDraw' that features image capture, drawing functionality, and the ability to save and share the resulting image.

> ## Try it out!
> 1. Clone the repo: `git clone https://github.com/Got17/Pic-draw-app.git`
> 2. Change directory to the project: `cd Pic-draw-app/PicDrawApp`
> 3. Build the project: `dotnet build` and `npx vite build`
> 4. Dowload all necessary packages: `npm install` 
> 5. Run the project: `npx vite` (for Web Browser) or `npx cap open android` (for Android device or emulator)

## 1. Setting Up Your Development Environment

To develop this app, you will need the following tools:

1. **Visual Studio**: Install Visual Studio with F# and WebSharper templates.
2. **Capacitor**: [Capacitor](https://capacitorjs.com "Capacitor") is a tool that helps you deploy web-based apps to Android and iOS. In order to use Capacitor successfully, please install Capacitor Core plugin (`npm i @capacitor/core`) and Capacitor command-line interface (CLI) (`npm i @capacitor/cli`).
3. **Android SDK**: Install the [Android SDK](https://developer.android.com/studio "Android SDK") for deploying the app to your device or emulator.
4. **WebSharper**: [WebSharper](https://websharper.com "WebSharper") helps you write full-stack applications in F#, including both frontend and backend.

## 2. Building PicDraw

Let’s break down the features of PicDraw:

#### a. User Interface

The app's interface consists of two main buttons:

- **Capture Image:** This button allows users to take a picture or select one from their gallery.
- **Save & Share:** This button enables users to save the annotated image and share it with others.

The code below shows the UI layout of the app, featuring the two buttons.

```html
<div id="main" ws-children-template="PicDraw">
    <div class="capture-button">
        <button id="captureBtn" ws-onclick="CaptureBtn">Capture Image</button>
    </div>
    <br />
    <div class="canvas-container">
        <canvas id="annotationCanvas" width="300" height="400" ws-hole="canvas"
                ws-onmouseDown="canvasMouseDown" ws-onmouseup="canvasMouseUp" 
                ws-onmouseout="canvasMouseOut" ws-onmousemove="canvasMouseMove"
                ws-onafterrender="canvasInit"
                ></canvas>
    </div>
    <div class="save-share-button">
        <button id="saveShareBtn" ws-onclick="SaveShareBtn">Save & Share</button>
    </div>
</div>
```

#### b. Capturing or Choosing an Image

To capture or choose an image, we use the [Camera plugin](https://capacitorjs.com/docs/apis/camera "Camera plugin") from Capacitor.

In order to use Camera plugin we need to install [PWA Elements](https://capacitorjs.com/docs/web/pwa-elements "PWA Elements") and add the script tags in your `index.html`.

```html
<script
  type="module"
  src="https://unpkg.com/@ionic/pwa-elements@latest/dist/ionicpwaelements/ionicpwaelements.esm.js"
></script>
<script
  nomodule
  src="https://unpkg.com/@ionic/pwa-elements@latest/dist/ionicpwaelements/ionicpwaelements.js"
></script>
```

The `takePicture()` function captures an image using the device’s camera or allows the user to select an image from their gallery. Once selected, the image is displayed on the canvas for drawing.

```fsharp
// Create canvas() function. We will use it only when it is needed.
let canvas() = As<HTMLCanvasElement>(JS.Document.GetElementById("annotationCanvas"))

// Function for putting the image on the canvas
let loadImageOnCanvas (imagePath: string) =
    let img = 
        Elt.img [
            on.load (fun img e ->
                let canvas = canvas()
                let ctx = canvas.GetContext "2d"
                ctx.ClearRect(0.0, 0.0, canvas.Width |> float, canvas.Height |> float)
                ctx.DrawImage(img, 0.0, 0.0, canvas.Width |> float, canvas.Height |> float)
            )
        ] []

    img.Dom.SetAttribute("src", imagePath)

// Function for taking or choosing a picture
let takePicture() = promise {
    let! image = Capacitor.Camera.GetPhoto(Camera.ImageOptions(
        resultType = Camera.CameraResultType.Uri,
        Source = Camera.CameraSource.PROMPT,
        Quality = 90
    ))
    image.WebPath |> loadImageOnCanvas
}
```

This functionality is triggered when the user clicks the `Capture Image` button. The image is then displayed on the canvas for further interaction. 

#### c. Drawing on the Image

Once the image is loaded onto the canvas, the user can draw annotations on it. Here’s how we handle drawing with both mouse and touch events:

```fsharp
// Get the canvas 2D context from the event target (the canvas element)
let getContext (e: Dom.EventTarget) = As<HTMLCanvasElement>(e).GetContext("2d")

// Function for drawing line on the canvas
let draw (e: Dom.EventTarget, offsetX, offsetY) =
    let ctx = getContext e
    ctx.StrokeStyle <- "#FF0000" 
    ctx.LineWidth <- 2.0 
    ctx.BeginPath()
    ctx.MoveTo(lastX.Value, lastY.Value)
    ctx.LineTo(offsetX, offsetY)
    ctx.Stroke()
    Var.Set lastX <| offsetX
    Var.Set lastY <| offsetY
```

In the above function, the drawing is handled by capturing mouse movements and drawing lines between two points on the canvas. The user can start drawing by clicking or tapping on the canvas, and the strokes will follow their movements.

```fsharp
let isDrawing = Var.Create false
let lastX, lastY = Var.Create 0.0, Var.Create 0.0 

let MouseUpAndOutAction (isDrawing) = 
    Var.Set isDrawing <| false  

// Template for Mouse events handling
IndexTemplate.PicDraw()
    .canvasMouseDown(fun e ->
        Var.Set isDrawing <| true
        Var.Set lastX <| (e.Event.OffsetX)
        Var.Set lastY <| (e.Event.OffsetY)
    )
    .canvasMouseUp(fun _ -> 
        MouseUpAndOutAction(isDrawing)
    )
    .canvasMouseOut(fun _ ->
        MouseUpAndOutAction(isDrawing)
    )
    .canvasMouseMove(fun e -> 
        let offsetX = e.Event.OffsetX
        let offsetY = e.Event.OffsetY

        if isDrawing.Value then
            draw (e.Target, offsetX, offsetY)
    )
    .Doc()
|> Doc.RunById "main"   
```

For touch support, the app listens for touch events and translates them into canvas drawing operations, ensuring a smooth drawing experience on mobile devices. Since there are no default values for `OffsetX` and `OffsetY` for touch events, we need to manually calculate those values.

```fsharp
// Calculate OffsetX 
let setOffsetX (touch: Touch) = 
    let canvas = canvas()
    let rect = canvas.GetBoundingClientRect()

    let scaleX = intTofloat(canvas.Width) / rect.Width

    let offsetX = (touch.ClientX - rect.Left) * scaleX

    offsetX // return offsetX

// Calculate OffsetY
let setOffsetY (touch: Touch) = 
    // Codes are similar to setOffsetX function...
    
    offsetY // return offsetY

// Template for Touch events handling
IndexTemplate.PicDraw()
    .canvasInit(fun () ->
        let canvas = canvas()
        canvas.AddEventListener("touchstart", fun (e: Dom.Event) -> 
            let touchEvent = e |> As<TouchEvent>
            touchEvent.PreventDefault()

            Var.Set isDrawing <| true

            let touch = touchEvent.Touches[0]

            let offsetX = setOffsetX(touch)
            let offsetY = setOffsetY(touch)

            Var.Set lastX <| offsetX
            Var.Set lastY <| offsetY
        )

        canvas.AddEventListener("touchmove", fun (e: Dom.Event) -> 
            let touchEvent = e |> As<TouchEvent>
            touchEvent.PreventDefault()

            let touch = touchEvent.Touches[0]

            let offsetX = setOffsetX(touch)
            let offsetY = setOffsetY(touch)

            if isDrawing.Value then
                draw(e.Target, offsetX, offsetY)
        )

        canvas.AddEventListener("touchend", fun (e: Dom.Event) -> 
            let touchEvent = e |> As<TouchEvent>
            touchEvent.PreventDefault()
            Var.Set isDrawing <| false
        )
    )
    .Doc()
|> Doc.RunById "main"  
```

#### d. Saving and Sharing the Image

Once the user has finished annotating the image, they can save and share it using the `Save & Share` button. The app uses [Filesystem](https://capacitorjs.com/docs/apis/filesystem "Filesystem plugin") and [Share](https://capacitorjs.com/docs/apis/share "Share plugin") plugins, which are part of Capacitor, to save the canvas image and share it via email, messaging, or other installed apps.

For Filesystem plugin, it is crucial to add permissions in order to access the files/folders. The following permissions must be added to your `.\android\app\src\main\AndroidManifest.xml` for Android.

```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

The function below allows the user to save the canvas as an image file and share it using the device's sharing options, making the app more interactive and useful for creating and sharing personalized images.

```fsharp
let canvas() = As<HTMLCanvasElement>(JS.Document.GetElementById("annotationCanvas"))

// Save and Share the result image 
let saveAndShareImage () = promise {
    let date = new Date()
    let canvas = canvas() 
    let fileName = $"{date.GetTime()}_image.png"
    let imageData = canvas.ToDataURL("image/png")
    let! savedImage = Capacitor.Filesystem.WriteFile(Filesystem.WriteFileOptions(
        Path = fileName,
        Data = imageData,                                
        Directory = Filesystem.Directory.DOCUMENTS

    ))

    Capacitor.Share.Share(Share.ShareOptions(
        Title = "Check out my annotated picture!",
        Text = "Here is an image I created using PicNote!",
        Url = savedImage.Uri,
        DialogTitle = "Share your creation"
    )) |> ignore

    return savedImage
}   
```

## 3. Testing and Deploying the App

Once all the features are in place, it’s time to test and deploy the app on an Android device:

**1. Build the WebSharper app:** Use Visual Studio or command line to compile the F# code.

**2. Integrate with Capacitor:** Initialize Capacitor in the project by using the command `npx cap sync` to sync your project with Android app and prepare for deployment.

**3. Deploy the app to Android:** Use the command `npx cap open android` to open the project in Android Studio and deploy the app to an Android device or emulator.

## Conclusion
The PicDraw App demonstrates how easy it is to build a functional Android app using F# and WebSharper. With features like image capture, annotation, and sharing, this app highlights the simplicity and power of F# for mobile development. If you’re interested in extending the app, consider adding features like color selection for drawing or more advanced image editing options.

By following this guide, you’ve gained hands-on experience with F# and WebSharper, enabling you to create more complex apps in the future. Happy coding!