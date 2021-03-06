---
title: "WebSharper Binding for Leaflet released"
categories: "leaflet,openstreetmap,f#,websharper"
abstract: "We are happy to announce the release of a WebSharper binding for Leafletjs, a library that makes it easy to display and work with maps such as OpenStreetMap."
identity: "3867,77204"
---
We are happy to announce the release of a WebSharper binding for Leafletjs, a library that makes it easy to display and work with maps such as OpenStreetMap. It is available [on NuGet](http://www.nuget.org/packages/WebSharper.Leaflet), and the sources are [on Github](https://github.com/intellifactory/websharper.leaflet).

Here is a sample to show you how easy it is to work with Leaflet in WebSharper:

```fsharp
Div [Attr.Style "height: 600px;"]
|>! OnAfterRender (fun div ->

    // Create the map
    let map = Leaflet.Map(div.Body)

    // Add the tiles from OpenStreetMap.
    // We provide the URL templates and copyright attributions for OpenStreetMap
    // and Mapbox as constants, so you don't waste time looking them up.
    map.AddLayer(
        Leaflet.TileLayer(
            Leaflet.TileLayer.OpenStreetMap.UrlTemplate,
            Leaflet.TileLayer.Options(
                Attribution = Leaflet.TileLayer.OpenStreetMap.Attribution)))

    // Set the initial view (latitude, longitude and zoom).
    map.SetView((47.49883, 19.0582), 14)

    // Add a marker at the position of our office.
    map.AddLayer(
        let m = Leaflet.Marker((47.4952, 19.07114))
        // Show a popup when the marker is clicked.
        m.BindPopup("IntelliFactory")
        m)

    // Add events to show the mouse position in another div named `coordinates'.
    map.On_mousemove(fun map ev ->
        coordinates.Text <- "Position: " + ev.Latlng.ToString())
    map.On_mouseout(fun map ev ->
        coordinates.Text <- "")
)
```

A meatier sample on the WebSharper website will follow soon. Happy coding!
