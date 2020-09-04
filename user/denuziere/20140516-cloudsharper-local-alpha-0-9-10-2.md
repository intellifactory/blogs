---
title: "CloudSharper Local alpha 0.9.10.2"
categories: "ide,cloudsharper,cloud,f#,websharper"
abstract: "This minor release of CloudSharper alpha focuses on a few long-standing bug fixes, as well as the addition of the \"Search and replace\" functionality in the editor."
identity: "3863,77196"
---
This minor release of CloudSharper alpha focuses on a few long-standing bug fixes, as well as the addition of the "Search and replace" functionality in the editor.

The communication protocol has been altered slightly, so don't forget to update your local component!

Here is the full change log:

 * [#366](https://bitbucket.org/IntelliFactory/cloudsharper/issue/366/changing-webserverport-in): Correctly propagate webserverport from CloudSharper.Console.exe.config.

 * [#372](https://bitbucket.org/IntelliFactory/cloudsharper/issue/372/newly-created-workspaces-dont-show-up-in): Refresh workspace list after creating a workspace.

 * [#373](https://bitbucket.org/IntelliFactory/cloudsharper/issue/373/no-ui-feedback-about-whether-projects-are): Show messages in build window when projects are first loaded. This should hopefully dissipate confusion when trying to build immediately after opening a workspace and the message "Error: project file invalid" showed up.

 * [#375](https://bitbucket.org/IntelliFactory/cloudsharper/issue/375/editor-add-ctrl-f-search): Add CodeMirror search. When the editor is focused, you can now search and/or replace text inside it. Here is the list of keyboard shortcuts:

    * `Ctrl+F` (`Cmd+F` on mac): start a search.
    * `Ctrl+G` (`Cmd+G` on mac): find the next match for the current search.
    * `Shift+Ctrl+G` (`Shift+Cmd+G` on mac): find the previous match for the current search.
    * `Shift+Ctrl+F` (`Cmd+Option+F` on mac): replace interactively.
    * `Shift+Ctrl+R` (`Shift+Cmd+Option+F` on mac): replace all.

 * [#377](https://bitbucket.org/IntelliFactory/cloudsharper/issue/377/add-forgot-your-password-form): Add "Forgot your password?" form.

Happy coding!
