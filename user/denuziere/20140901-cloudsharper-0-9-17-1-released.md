---
title: "CloudSharper 0.9.17.1 released"
categories: "cloudsharper,f#,websharper"
abstract: "This release enables running F# interactive without a workspace, in addition to a set of bug fixes."
identity: "4014,77379"
---
It's release time again! CloudSharper 0.9.17.1 is out with a new feature: workspace-less fsi.

You can now run F# Interactive without having a workspace open, which is quite convenient for quick exploratory programming. Note that your fsi session will be reset as soon as you open a workspace.

You can also dock the interactive window to the right of the main tabset, making it easier to work interactively with a script or a documentation page.

[![](http://i.imgur.com/4ks9yyQ.png)](http://i.imgur.com/NPrQKRo.png)

This release also fixes a critical issue where the MSBuild agent on mono would throw an exception when trying to start a build.

As usual, here is the complete change log:

 * [#467](https://bitbucket.org/IntelliFactory/cloudsharper/issue/467/check-subprotocol-for-globalcommands-too): Check subprotocol for GlobalCommands too
 * [#474](https://bitbucket.org/IntelliFactory/cloudsharper/issue/474/line-endings-in-console-for-foreign): Fix newlines in console output of foreign commands
 * [#478](https://bitbucket.org/IntelliFactory/cloudsharper/issue/478/show-the-cs-version-number-on-the): Show the CloudSharper version number on the dashboard
 * [#490](https://bitbucket.org/IntelliFactory/cloudsharper/issue/490/enable-fsi-in-workspace-less-scenarios): Enable fsi in workspace-less scenarios
 * [#499](https://bitbucket.org/IntelliFactory/cloudsharper/issue/499/select-current-file-in-sln-and-files-tree): Add "Find in solution explorer" and "Find in file explorer" to tab context menu
 * [#503](https://bitbucket.org/IntelliFactory/cloudsharper/issue/503/dialog-layout-on-viewport-resize): Dialog layout on viewport resize
 * [#504](https://bitbucket.org/IntelliFactory/cloudsharper/issue/504/workspace-create-a-workspace-does-nothing): Fix the `new-ws` command
 * [#505](https://bitbucket.org/IntelliFactory/cloudsharper/issue/505/fpish-account-association-page-style): Fix styling of the FPish account association page
 * [#507](https://bitbucket.org/IntelliFactory/cloudsharper/issue/507/the-msbuild-agent-throws-an-exception-on): Fix exception thrown by the MSBuild agent on mono when triggering a build
 * [#509](https://bitbucket.org/IntelliFactory/cloudsharper/issue/509/websocket-server-refuses-connections-from): Correctly connect to a local component running on another machine
 * [#510](https://bitbucket.org/IntelliFactory/cloudsharper/issue/510/allow-docking-the-interactive-window): Allow docking the interactive window

Happy coding!
