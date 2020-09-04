---
title: "CloudSharper Local alpha 0.9.6"
categories: "cloudsharper,f#"
abstract: "In addition to several bug fixes, the main highlight of this new update is the experimental support for MSBuild as an alternative to IntelliFactory.Build."
identity: "3741,77067"
---
Here are the changes for CloudSharper Local alpha 0.9.6:

 * As Adam pre-announced [yesterday](/user/granicz/20140217-update-showing-compiler-errors.md), we now have experimental support for MSBuild as an alternative to IntelliFactory.Build. This includes:

    * A new `msbuild` console command that runs MSBuild. Like `build`, if you do not pass a build file as argument, it searches for the first compatible build file in the workspace (ie. `*.*proj` or `*.sln`).

    * Intellisense can now read source files and references from `*.fsproj` files. In particular, this partially fixes [#270](https://bitbucket.org/IntelliFactory/cloudsharper/issue/270/project-shows-errors-immediately-after): for projects with no other dependencies than F# and WebSharper, the completion service doesn't report errors anymore on valid projects even before building.

    * The keyboard shortcut `Ctrl+B` now triggers `msbuild` rather than `build`.

 * [#266](https://bitbucket.org/IntelliFactory/cloudsharper/issue/266/deleted-workspace-remains-listed): Unlist deleted workspaces.

 * [#268](https://bitbucket.org/IntelliFactory/cloudsharper/issue/268/keep-focus-in-active-tab-when-sending-code): Keep focus in the active tab when sending code to FSI.

 * [#269](https://bitbucket.org/IntelliFactory/cloudsharper/issue/269/delete-menu-should-not-be-available-at-the): Remove "Delete" context menu option from the workspace root.

 * [#271](https://bitbucket.org/IntelliFactory/cloudsharper/issue/271/deploy-menu-available-prior-to-build): When a Sitelets deployment fails, warn that it might be because the project wasn't built.

 * [#273](https://bitbucket.org/IntelliFactory/cloudsharper/issue/273/view-synchronization-issues): When deleting the current workspace, close it first.

 * [#275](https://bitbucket.org/IntelliFactory/cloudsharper/issue/275/feedback-mechanism): Clear the feedback window after a message has been sent.
 
 * [#278](https://bitbucket.org/IntelliFactory/cloudsharper/issue/278/appharbour-deployment-config-results-in): When deploying to AppHarbor, warn about the possibility that the browser might be blocking the popup.
 
 * [#279](https://bitbucket.org/IntelliFactory/cloudsharper/issue/279/intellisense-error-count-not-updating): Remove Intellisense errors related to deleted files.

Some future work will focus on the MSBuild support, in particular adding support for simply installing NuGet packages.
