---
title: "CloudSharper 0.9.13 with workspace cloud sharing!"
categories: "cloudsharper,f#,websharper"
abstract: "With this new release of CloudSharper, we are starting the move of CloudSharper towards more and more online publication and sharing capabilities."
identity: "3920,77259"
---
Here we are again with a new release of CloudSharper today. This is an exciting one, as it allows you to upload your workspaces to our cloud storage and share them with everyone!

To use this feature, simply go to the "Workspace" menu and choose "Publish Workspace", or type `sync up` in the console. The contents of the current workspace will be uploaded as shown below:

[![](http://i.imgur.com/nVw8nDZ.png)](http://i.imgur.com/rJZXm32.png)

Once uploaded, you can share your work in two different ways: as a downloadable zip, or as a link to clone it as a new local CloudSharper workspace. It's that easy!

In order to avoid publishing workspaces littered with generated and downloaded files, we support a `.csignore` configuration file in the root of the workspace. This file contains a list of glob patterns for files and directories that should not be included in the uploaded workspace. As an example, here is the `.csignore` we now include with every newly created workspace:

```
/packages/
bin/
obj/
```

One last detail about this feature: subsequent uploads of the same workspace will overwrite the current one, so you can re-run `sync up` to update the online version of your workspace. In a future release, we will also include the capability to upload a snapshot of the workspace (or just a subfolder) under a separate unique ID.

Here is the full change log for this release:

 * Implement `sync up` command, and corresponding menu entry, to upload the current workspace to Azure cloud storage via cloudsharper.com.
 * [#426](https://bitbucket.org/IntelliFactory/cloudsharper/issue/426/check-restore-csignore-functionality): Use `.csignore` file as a list of glob patterns for files that should not be included in uploaded workspace.
 * [#430](https://bitbucket.org/IntelliFactory/cloudsharper/issue/430/automatically-add-a-csignore-to-new): Automatically add a `.csignore` to new workspaces.
 * [#429](https://bitbucket.org/IntelliFactory/cloudsharper/issue/429/add-registration-facility-on-the-login): Simplified the login page to a more minimalist style and added a registration form on this page.
 * [#428](https://bitbucket.org/IntelliFactory/cloudsharper/issue/428/put-zip-download-behind-authentication): Put `/zip` download behind authentication.
 * NuGet source settings, including credentials, are now saved in a local file (`~/CloudSharper/<username>/Workspaces/NuGet.config`).
 * Editor tabs now have a context menu for saving or closing tabs.
 * Added Disqus comment threads to our blog entries.
