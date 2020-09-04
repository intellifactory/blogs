---
title: "CloudSharper 0.9.17 released"
categories: "cloudsharper,f#,websharper"
abstract: "This is a minor bugfix release."
identity: "4010,77373"
---
We just published a bugfix release to CloudSharper Local alpha. This bugfix slightly alters the communication protocol between the web application and the installed component, which is why the version number was bumped to 0.9.17 and upgrading is required.

Change log:

 * [#492](https://bitbucket.org/IntelliFactory/cloudsharper/issue/492/do-not-open-readme-md-only-readme-md): On workspace open, only open README.*.md
 * [#500](https://bitbucket.org/IntelliFactory/cloudsharper/issue/500/delete-doesnt-work-properly): Silent error when deleting a directory
 * [#501](https://bitbucket.org/IntelliFactory/cloudsharper/issue/501/workspaces-with-websharper-dont-close): Can't delete an open workspace
 * [#502](https://bitbucket.org/IntelliFactory/cloudsharper/issue/502/nuget-reference-versioning-on-adding-a): NuGet reference versioning on adding a package

Happy coding!
