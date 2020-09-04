---
title: "WebSharper 3.0.3-alpha released"
categories: "javascript,f#,websharper"
abstract: "This is our first release of WebSharper 3.0-alpha with a new feature: source mapping."
identity: "4146,77594"
---
This is our first release of WebSharper 3.0-alpha with a new feature: source mapping. If you enable it on your project, you can see the F# sources in your browser and set breakpoints in it.

![](https://i.imgur.com/IvLimDc.png)


## Source maps

* You can enable including source maps and the required source files in a WebSharper assembly, by adding the

    ```xml
        <WebSharperSourceMap>True</WebSharperSourceMap>
    ```

    property to your project file.

 * To unpack the source maps in your web project, add this same property to the project file, and

    ```xml
        <staticContent>
            <mimeMap fileExtension=".fs" mimeType="text/plain" />
        </staticContent>
    ```

    inside the `web.config` file's `<system.webServer>` element.

 * It is also recommended to set

    ```xml
        <OtherFlags>--quotations-debug</OtherFlags>
    ```

    to have source position information inside reflected definitions available for WebSharper. Otherwise only the starting lines of functions can be mapped.

 * Single-Page Application projects are currently not supported.

 * In Google Chrome, you need to check the "Enable JavaScript source maps" setting in Developer Tools Settings.

 * For Internet Explorer, you need to have Windows 8.1 Update 1.

 * `websharper.exe` also has new flag `-sm` for source map embedding/unpacking.


## Sitelets improvements

 * We added combinators to simplify writing Sitelets for sub-actions and embedding them into Sitelets for wider actions. See [issue #307](https://github.com/intellifactory/websharper/issues/307) for more details about these combinators.


## WIG improvements

 * Added `ObsoleteWithMessage` function, with it you can specify a message for the obsolete warning.
