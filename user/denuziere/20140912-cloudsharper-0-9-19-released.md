---
title: "CloudSharper 0.9.19 released"
categories: "cloudsharper,f#,websharper"
abstract: ""
identity: "4029,77413"
---
CloudSharper 0.9.19 is out! Here are some highlights:

 * Workspaces are now capable of managing zero, one or several solutions, instead of just one. No solution file is created when you first create a workspace, but it can be automatically created when you create a project containing a project file; this behavior is customized by the combobox in the "New project" window.

   [![](http://i.imgur.com/UjmxMPw.png)](http://i.imgur.com/UjmxMPw.png)

 * Configuration of listening urls and ports has slightly changed, both in `CloudSharper.Console.exe.config` if you use the console and on the `CloudSharper.exe` command line options. There is now a `serveip` to decide on which IP the various servers (Websockets, static files, sitelets) should listen, and a `*port` option for each of these servers.

Here is the full change log:

 * [#444](https://bitbucket.org/IntelliFactory/cloudsharper/issue/444/support-multiple-solution-files-in-a): Support multiple solution files in a workspace
 * [#484](https://bitbucket.org/IntelliFactory/cloudsharper/issue/484/missing-files-should-be-marked-in-solution): Missing files marked in the Solution view
 * [#514](https://bitbucket.org/IntelliFactory/cloudsharper/issue/514/unable-to-add-existing-files-to-project): Problem adding existing files to a project
 * [#515](https://bitbucket.org/IntelliFactory/cloudsharper/issue/515/rename-project-item-should-update-proj): Update project file when renaming a project item
 * [#516](https://bitbucket.org/IntelliFactory/cloudsharper/issue/516/loading-custom-editors-stays-on-forever-if): "Loading custom editors" stays on forever if the user has custom editors without js dependencies
 * [#517](https://bitbucket.org/IntelliFactory/cloudsharper/issue/517/cloudsharper-workspace-filename-do-not): CloudSharper workspace filename do not refresh
 * [#518](https://bitbucket.org/IntelliFactory/cloudsharper/issue/518/solution-less-workspaces): Support no solution files in a workspace
 * [#523](https://bitbucket.org/IntelliFactory/cloudsharper/issue/523/nuget-dll-reference-gets-removed-when): NuGet dll reference gets removed when updating another package
 * [#525](https://bitbucket.org/IntelliFactory/cloudsharper/issue/525/the-new-project-dialog-fails-to-show-up): The New Project... dialog fails to show up
 * [#526](https://bitbucket.org/IntelliFactory/cloudsharper/issue/526/cloudsharper-service-exception-on-start): Fix detecting http server failure to listen
 * [#527](https://bitbucket.org/IntelliFactory/cloudsharper/issue/527/project-file-updating): Project file updating in solution tree

Happy coding!
