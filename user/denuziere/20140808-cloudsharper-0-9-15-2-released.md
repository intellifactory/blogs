---
title: "CloudSharper 0.9.15.2 released"
categories: "cloudsharper,f#,websharper"
abstract: ""
identity: "3995,77358"
---
We just released CloudSharper 0.9.15.2, and as you probably noticed, there have been quite big visual changes with the 0.9.15 series as we switch to Dojo as our UI toolkit.

[![](http://i.imgur.com/uz7b7u9.png)](http://i.imgur.com/PW8Sf5P.png)

[![](http://i.imgur.com/VHUmsAF.png)](http://i.imgur.com/1fGMEe0.png)

## Change log

Besides this visual update, here are the recent changes:

 * [#450](https://bitbucket.org/IntelliFactory/cloudsharper/issue/450/): No more unneeded file change notifications when opening a file shortly after opening a workspace.
 * [#461](https://bitbucket.org/IntelliFactory/cloudsharper/issue/461/type-checking-is-very-slow-to-kick-in): Cancel code completion request when another is sent or the tab is switched.
 * [#464](https://bitbucket.org/IntelliFactory/cloudsharper/issue/464/report-error-when-moving-a-file-folder): Report error when moving a file/folder failed.
 * [#465](Implement file/directory copying): Implement file/directory copying. This can be done either using the `cp` command in the console, or using Ctrl+drag and drop in the file tree.
 * Initial login should now be faster.

## Troubleshooting

 * If you experience a slow loading of the solution tree, be sure to dowload the latest installer.

 * If on startup, the local component shows the following error followed by a stack trace:

    ```
    fsharpSystem.TypeInitializationException: The type initializer for
    '<StartupCode$CloudSharper-Backend-Local>.$LocalBackend' threw an exception.
    ---> System.IO.FileNotFoundException: Could not load file or assembly
    'IntelliFactory.IO, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null'
    or one of its dependencies. The system cannot find the file specified.
    ```

    Then you need to run the installer again and select "Repair"; things should run smoothly from there.

Happy coding!
