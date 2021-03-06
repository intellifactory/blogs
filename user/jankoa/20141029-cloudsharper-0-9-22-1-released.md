---
title: "CloudSharper 0.9.22.1 released"
categories: "cloudsharper,f#,websharper"
abstract: "This is a bugfix release for cloud workspace management."
identity: "4076,77505"
---
This is a bugfix release for cloud workspace management. A context menu item was added on cloud workspaces to get the clone and download links. If you delete a cloud workspace, you still have to wait a minute before you can re-upload it, but get correct notification about the failed upload.

Full change log:

 * [#571](https://bitbucket.org/IntelliFactory/cloudsharper/issue/571/): Download zip returns a corrupt archive.
 * [#569](https://bitbucket.org/IntelliFactory/cloudsharper/issue/569/): Add 'Get Share URL' option to cloud workspaces on the dashboard.
 * [#570](https://bitbucket.org/IntelliFactory/cloudsharper/issue/570/): Sync up fails if there are spaces in the workspace name.
 * [#574](https://bitbucket.org/IntelliFactory/cloudsharper/issue/574/): Dashboard workspace menus are not updated after sync operations.
 * [#566](https://bitbucket.org/IntelliFactory/cloudsharper/issue/566/): Fix uploading a previously deleted cloud workspace.

Happy coding!
