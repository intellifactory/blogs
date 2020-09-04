---
title: "CloudSharper 0.9.22.2 released"
categories: "cloudsharper,f#,websharper"
abstract: "This is a bugfix release with some additions for FSI."
identity: "4100,77530"
---
This is a bugfix release with some additions for FSI.

![](https://i.imgur.com/gwFx92X.png)

Full changes:

 * [#568](https://bitbucket.org/IntelliFactory/cloudsharper/issue/568/): Do not send multiple commands to local service when using the "Retry" button on the dashboard screen.
 * Markdown documents with embedded F# code blocks show two buttons now. "Send to interactive" was renamed to "Run" and "Edit" just sends the contents to the Interactive input box without executing it.
 * DLL files have a "Reference in interactive" context menu which runs an #r command with that library.
 * "Find in solution explorer" context menu item on tab headers is only visible when the file has a node in solution view.

Happy coding!
