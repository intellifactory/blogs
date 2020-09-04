---
title: "CloudSharper 0.9.14 released"
categories: "cloudsharper,f#,websharper"
abstract: "This release is geared towards enhanced usability with two main features: a reworked dashboard and a new solution view."
identity: "3929,77269"
---
We just released version 0.9.14 of CloudSharper. This release is mainly geared towards enhanced usability with two main features:

 * A reworked dashboard, replacing both the old "Go to CloudSharper" page and the workspace list window with a streamlined page containing all the information you need.

   [![](http://i.imgur.com/gy99All.png)](http://i.imgur.com/6cFoRvA.png)

 * A redesign of the left panel of the workspace view. Instead of the file system tree, now a logical solution/project tree is the default view. All functionality related to project handling has been moved to the context menus here: adding references, managing NuGet packages, building and cleaning; managing the list of files compiled or included in a project has been implemented. Note that file create, rename and delete are still available in the Files panel, but do not affect the list of files included in the project.

   [![](http://i.imgur.com/cL7fhhF.png)](http://i.imgur.com/Rj3gZUS.png)

Here is a complete change log:

 * [#402](https://bitbucket.org/IntelliFactory/cloudsharper/issue/402/show-a-vs-like-project-list): Show a VS-like project list.
 * [#411](https://bitbucket.org/IntelliFactory/cloudsharper/issue/411/warn-on-file-changes): Alert the user about changes to open files.
 * [#422](https://bitbucket.org/IntelliFactory/cloudsharper/issue/422/sort-available-deployment-options): Sort available deployment options.
 * [#437](https://bitbucket.org/IntelliFactory/cloudsharper/issue/437/nugetinstallpackage-in-fsi-uses-the-wrong): NuGet.InstallPackage in FSI uses the wrong Nuget.Core.dll
 * Replace "No local component" error window with a flash on the dashboard.
 * Use Gravatar to retrieve the user's picture.

Happy coding!
