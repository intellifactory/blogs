---
title: "CloudSharper Local alpha 0.9.10.1 released"
categories: "ide,cloudsharper,cloud,f#,websharper"
abstract: ""
identity: "3845,77178"
---
[CloudSharper 0.9.10.1](http://cloudsharper.com) is here! It has been several weeks since the previous release, therefore this version includes several major improvements.

 * CloudSharper is now **officially supported on mono**. Here are the installation instructions for Ubuntu 14.04:

    * Install the following packages from the Ubuntu repository: `mono-complete fsharp`

    * Download the zip version of CloudSharper Local, available [here](http://cloudsharper.com/downloads/CloudSharper.zip) (or linked on the login page) and extract it in the folder of your choice.

    * To start CloudSharper Local, run the following command: `mono CloudSharper.exe`

 * The **F# Intellisense service** is now based on the latest [FSharp.Compiler.Service](https://github.com/fsharp/FSharp.Compiler.Service), the community-maintained library used by the MonoDevelop and Emacs modes for F#.
 
 * The **F# Interactive service** is also based on FSharp.Compiler.Service. This gives us more control over the internals of fsi and allows more reliable extensibility for tools such as our inline charting library and on-the-fly compilation to JavaScript via WebSharper.
 
 * CloudSharper now manages a **solution file** per workspace. It is a standard Visual Studio solution file, allowing you to work comfortably on the same workspace with CloudSharper and Visual Studio. Projects are automatically added to it when created with the "Create new project" window, and removed when the `.*proj` file is deleted.
 
 * Minimal **NuGet package management**. CloudSharper checks for any `packages.config` file in your workspace and automatically installs the corresponding packages. Upcoming versions will provide automatic insertion of the relevant references in project files and a GUI to easily search, install and reference NuGet packages.

The following long-standing bugs have been fixed:

 * On some computers, the CloudSharper Local Console failed to start properly, which forced these users to start `CloudSharper.exe` from the command line. This is not the case anymore.

 * The installer now properly removes older versions of CloudSharper Local.

Here is the complete change list:

 * [#139](https://bitbucket.org/IntelliFactory/cloudsharper/issue/139/exit-1-in-f-interactive-gives-error): "exit 1;;" in F# Interactive gives error message
 * [#272](https://bitbucket.org/IntelliFactory/cloudsharper/issue/272/console-doesnt-start-service-properly): Console doesn't start service properly
 * [#305](https://bitbucket.org/IntelliFactory/cloudsharper/issue/305/projects-not-in-subfolders-crash-the): Projects not in subfolders crash the console
 * [#308](https://bitbucket.org/IntelliFactory/cloudsharper/issue/308/cloudsharperinstallermsi-is-not-removing): CloudSharper.Installer.msi is not removing older versions
 * [#309](https://bitbucket.org/IntelliFactory/cloudsharper/issue/309/use-fsi-from-fsharpcompilerservice-instead): Use fsi from FSharp.Compiler.Service instead of the current wrapper
 * [#313](https://bitbucket.org/IntelliFactory/cloudsharper/issue/313/context-menu-on-workspaces-list): Context menu on workspaces list
 * [#316](https://bitbucket.org/IntelliFactory/cloudsharper/issue/316/do-not-ship-systemdll-microsoftbuild-etc): Do not ship System.dll, Microsoft.Build.* etc - breaks on Mono
 * [#318](https://bitbucket.org/IntelliFactory/cloudsharper/issue/318/fsharpcore-4300-vs-4310): FSharp.Core 4.3.0.0 vs 4.3.1.0
 * [#319](https://bitbucket.org/IntelliFactory/cloudsharper/issue/319/conflicts-between-ms-build-40-and-120): Conflicts between MS Build 4.0 and 12.0
 * [#323](https://bitbucket.org/IntelliFactory/cloudsharper/issue/323/mono-stuck-at-creating-interactive-service): Mono; stuck at creating Interactive service
 * [#324](https://bitbucket.org/IntelliFactory/cloudsharper/issue/324/mono-downloadtemplate-hits-notimplemented): Mono: downloadTemplate hits NotImplemented
 * [#325](https://bitbucket.org/IntelliFactory/cloudsharper/issue/325/start-csexe-bug): Start cs.exe bug
 * [#326](https://bitbucket.org/IntelliFactory/cloudsharper/issue/326/login-with-fpish-redirect-needs-updating): Login with FPish redirect needs updating
 * [#327](https://bitbucket.org/IntelliFactory/cloudsharper/issue/327/file-tabs-improvements): File tabs improvements
 * [#329](https://bitbucket.org/IntelliFactory/cloudsharper/issue/329/solution-file-support): Solution file support
 * [#330](https://bitbucket.org/IntelliFactory/cloudsharper/issue/330/project-references-dialog): Project references dialog
 * [#331](https://bitbucket.org/IntelliFactory/cloudsharper/issue/331/closing-affected-file-tabs-on-folder): Closing affected file tabs on folder delete
 * [#332](https://bitbucket.org/IntelliFactory/cloudsharper/issue/332/asyncdisposable-losing-exceptions): AsyncDisposable losing exceptions.
 * [#333](https://bitbucket.org/IntelliFactory/cloudsharper/issue/333/bug-in-portrangeallocator-asyncdisposable): bug in portRangeAllocator + AsyncDisposable
 * [#334](https://bitbucket.org/IntelliFactory/cloudsharper/issue/334/fsi-gets-started-twice-per-session): FSI gets started twice per session
 * [#335](https://bitbucket.org/IntelliFactory/cloudsharper/issue/335/weird-behavior-in-fsi-values-not-printed), [#337](https://bitbucket.org/IntelliFactory/cloudsharper/issue/337/mono-fsi-does-not): Mono: fsi does not print anything to the console
 * [#336](https://bitbucket.org/IntelliFactory/cloudsharper/issue/336/cloudsharperexe-spinning-cpu): CloudSharper.exe spinning CPU
 * [#339](https://bitbucket.org/IntelliFactory/cloudsharper/issue/339/msbuild-building-cleaning-projects-fails): MSBuild building/cleaning projects fails on Mono
 * [#341](https://bitbucket.org/IntelliFactory/cloudsharper/issue/341/executing-an-fsi-script-fails-with): Executing an FSI script fails with fireworks on Mono
 * [#342](https://bitbucket.org/IntelliFactory/cloudsharper/issue/342/msbuild-service-find-a-solution-that-works): MSBuild service -- find a solution that works both on Mono and .NET
 * [#343](https://bitbucket.org/IntelliFactory/cloudsharper/issue/343/allow-creating-an-empty-workspace-without): Allow creating an empty workspace without needing to go to azure
 * [#345](https://bitbucket.org/IntelliFactory/cloudsharper/issue/345/fsharpcore-v4310-not-found-on-mono): FSharp.Core v4.3.1.0 not found on Mono
 * [#346](https://bitbucket.org/IntelliFactory/cloudsharper/issue/346/mono-html-project-connection-reset-when): Mono / HTML project: Connection Reset when viewing generated HTML
 * [#347](https://bitbucket.org/IntelliFactory/cloudsharper/issue/347/mono-native-failure-on): Mono: native failure on FSharpInterfaceDataVersionAttribute ctor
 * [#349](https://bitbucket.org/IntelliFactory/cloudsharper/issue/349/completion-returns-nothing-when-the): Completion returns nothing when the project file has been opened
 * [#350](https://bitbucket.org/IntelliFactory/cloudsharper/issue/350/build-dependencies-missing): Build dependencies missing
 * [#351](https://bitbucket.org/IntelliFactory/cloudsharper/issue/351/completion-not-working-for-websharper): Completion not working for WebSharper projects (mono + windows)
 * [#352](https://bitbucket.org/IntelliFactory/cloudsharper/issue/352/completion-not-working-after-build-in-mono): Completion not working after build in Mono
 * [#353](https://bitbucket.org/IntelliFactory/cloudsharper/issue/353/add-the-latest-nugetexe-to-the-installer): Add the latest nuget.exe to the installer and zip
 * [#354](https://bitbucket.org/IntelliFactory/cloudsharper/issue/354/execute-nuget-package-restore-when): Execute NuGet package restore when necessary
 * [#355](https://bitbucket.org/IntelliFactory/cloudsharper/issue/355/add-a-nuget-package-restore-context-menu): Add a "NuGet Package Restore" context menu item
 * [#357](https://bitbucket.org/IntelliFactory/cloudsharper/issue/357/package-restore-output-should-go-into): "Package restore" output should go into Build Messages
 * [#358](https://bitbucket.org/IntelliFactory/cloudsharper/issue/358/fsproj-changes-not-getting-picked-up): fsproj changes not getting picked up
 * [#363](https://bitbucket.org/IntelliFactory/cloudsharper/issue/363/triggering-msbuild-when-another-build-is): Triggering msbuild when another build is already running crashes the local component
 * [#364](https://bitbucket.org/IntelliFactory/cloudsharper/issue/364/msbuild-service-throws-an-exception-if-a): MSBuild service throws an exception if a project file is invalid
 * [#365](https://bitbucket.org/IntelliFactory/cloudsharper/issue/365/msbuild-is-silent-when-passed-an-invalid): MSBuild is silent when passed an invalid project file
 * [#367](https://bitbucket.org/IntelliFactory/cloudsharper/issue/367/fsi-websharper-code-raises-failed-to): fsi: WebSharper code raises Failed to translate property access
 * [#370](https://bitbucket.org/IntelliFactory/cloudsharper/issue/370/mono-exception-when-writing-to-the): Mono: exception when writing to the solution file

Happy coding!
