---
title: "CloudSharper 0.9.19.3 released"
categories: "cloudsharper,f#,websharper"
abstract: "This is a bugfix release focusing mainly on the F# code service."
identity: "4038,77437"
---
The main highlight of this release is the switch from plain [FSharp.Compiler.Service](https://github.com/fsharp/fsharp.compiler.service) to the higher-level [FSharp.CompilerBinding](https://github.com/fsharp/fsharpbinding). This will allow us to benefit from more community-developed enhancements, and to develop improvements to the code service more quickly. A few bugs have already been fixed thanks to this transition, and you can expect hover tooltips, go to definition and more soon!

Full change log:

 * [#281](https://bitbucket.org/IntelliFactory/cloudsharper/issue/281/deleting-files-folders-should-close-any): Close affected tabs when running `rmdir`
 * [#300](https://bitbucket.org/IntelliFactory/cloudsharper/issue/300/no-ui-feedback-when-trying-to-run-an-exe): Always fail as expected when trying to run a program that doesn't exist from the console
 * [#543](https://bitbucket.org/IntelliFactory/cloudsharper/issue/543/auto-close-and-match-brackets-in-editor): Auto-close and match brackets in the editor
 * [#544](https://bitbucket.org/IntelliFactory/cloudsharper/issue/544/upload-file-window-style-botched): Fix upload file window style
 * [#546](https://bitbucket.org/IntelliFactory/cloudsharper/issue/546/tab-filename-reassignment-on-move-is): Reassign tab when the file's directory is renamed
 * [#547](https://bitbucket.org/IntelliFactory/cloudsharper/issue/547/hideorshowslntree-only-works-if-the): Fix persisting "Loading custom editors..." message in some cases
 * [#551](https://bitbucket.org/IntelliFactory/cloudsharper/issue/551/changes-in-files-are-not-visible-in): Correctly reflect saved changes to a file in the code service for another file of the same project
 * [#552](https://bitbucket.org/IntelliFactory/cloudsharper/issue/552/completion-list-doesnt-update-itself-when): Fix updating the completion list when typing extra letters

Happy coding!
