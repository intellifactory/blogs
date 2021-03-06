---
title: "WebSharper Extensions for Bing Maps 7.0 Updated"
categories: "bing,maps,websharper"
abstract: "The WebSharper Extensions for Bing Maps has been updated to reflect the last changes in the 7.0 API. Among notable additions are Directions, a simplified interface to the Routes REST API, Traffic and VenueMaps, as well as the possibility to implement your own extension modules."
identity: "2054,74886"
---
The WebSharper Extensions for Bing Maps has been updated to reflect the last changes in the 7.0 API. Here is a list of additions:

 * A new "modules" API which allows you to create extension modules for Bing Maps. This also comes with the three following built-in modules.

    * `Directions`: This is a simplified API to access the `Routes` REST API. It provides a very convenient interface, with attractive features such as draggable waypoints which trigger a recalculation of the route, and clickable instructions which causes the map to zoom on the corresponding location.
    * We advise you to only use our previously existing interface to the `Routes` REST API if you want to use it without displaying a map.
    * `Traffic`: This simple class allows you to display traffic information as a layer on the map.
    * `VenueMaps`: This API provides precise information about buildings surrounding an area, and draws their ground footprint on the map.

 * A new `strokeDashArray` property for polylines and polygons which allows you to use dashed lines. The WebSharper extension use an array of integers to represent the list of dash lengths more safely than the string used by the JavaScript API.



To demonstrate the simplicity of these new APIs, here is the code for a map allowing the user to search for a route between two locations.

```fsharp
open IntelliFactory.WebSharper.Bing
open IntelliFactory.WebSharper.Bing.Directions

[<JavaScript>]
let MapWithRouteRequest () =
    let origin = Input []
    let destination = Input []
    let button = Input [Attr.Type "button"; Attr.Value "Request route"]
    let answer = Div [Id "answer"]
    let mapContainer =
        Div []
        |>! OnAfterRender (fun el ->
            let opts = MapOptions(Credentials = "Your Bing maps developer key",
                                  Width = 600,
                                  Height = 500)
            let map = Map(el.Body, opts)
            map.SetMapType(MapTypeId.Road)
            let onDirsLoaded() =
                let dirman = DirectionsManager(map)
                let request (_:Element) (_:Events.MouseEvent) =
                    dirman.ResetDirections()
                    dirman.AddWaypoint
			            <| Waypoint(WaypointOptions(Address = origin.Value))
                    dirman.AddWaypoint
			            <| Waypoint(WaypointOptions(Address = destination.Value))
                    dirman.SetRenderOptions
			            <| DirectionsRenderOptions(ItineraryContainer = answer.Body)
                    dirman.SetRequestOptions
			            <| DirectionsRequestOptions(DistanceUnit = DistanceUnit.Kilometers)
                    dirman.CalculateDirections()
                button |>! OnClick request |> ignore
            Maps.LoadModule("Microsoft.Maps.Directions", LoadModuleArgs(onDirsLoaded))
        )
    Div [mapContainer
         Span[Text "From:"]; origin
         Span[Text "To:"]; destination
         button
         answer]
```

<img src=http://i.imgur.com/Dx0QP.png>

The updated extension for Bing Maps can be downloaded at <a href=http://websharper.com/extension/358-bmaps/Some/2.3.17>this address</a>.
