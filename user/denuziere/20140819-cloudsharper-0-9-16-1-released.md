---
title: "CloudSharper 0.9.16.1 released"
categories: "cloudsharper,f#,websharper"
abstract: "This is a minor bugfix release of CloudSharper Local alpha."
identity: "4009,77372"
---
We just released CloudSharper 0.9.16.1, a bugfix release fixing issues both in the GUI and the installed component. It is protocol-compatible, which means that you can still connect to CloudSharper using the 0.9.16.0 local component, but you will not benefit from some of the fixes listed below.

Here is the change log:

 * [#479](https://bitbucket.org/IntelliFactory/cloudsharper/issue/479/show-correct-icons-on-sln-tree-instead-of): Use better icons in the solution tree instead of folders.
 * [#481](https://bitbucket.org/IntelliFactory/cloudsharper/issue/481/close-warning-on-tabs-that-belong-to): Close/warning on tabs that belong to deleted files.
 * [#483](https://bitbucket.org/IntelliFactory/cloudsharper/issue/483/layout-issues-in-toolbar-on-deploy-and): Layout issues in toolbar on Deploy and Split view.
 * [#493](https://bitbucket.org/IntelliFactory/cloudsharper/issue/493/msbuild-using-cached-websharper-references): Isolate MSBuild in an AppDomain to prevent it from reusing the same WebSharper.Tasks.dll after switching workspaces.
 * [#494](https://bitbucket.org/IntelliFactory/cloudsharper/issue/494/ignore-empty-lines-and-comments-on-code): Ignore empty lines and comment on code folding.
 * [#496](https://bitbucket.org/IntelliFactory/cloudsharper/issue/496/add-nuget-sources-to-tools): Add "Tools > Nuget sources" menu entry.
 * [#497](https://bitbucket.org/IntelliFactory/cloudsharper/issue/497/enable-disable-key-bindings): Add "Tools > Keyboard shortcuts enabled" toggle menu entry.
 * [#498](https://bitbucket.org/IntelliFactory/cloudsharper/issue/498/add-a-better-icon-to-close-minimize-on-fsi): Add a better icon to close/minimize on fsi.

Happy coding!
