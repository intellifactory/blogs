---
title: "CloudSharper 0.9.16 released"
categories: "cloudsharper,f#,websharper"
abstract: "This release marks the first of many upcoming features to customize your environment."
identity: "3997,77361"
---
CloudSharper 0.9.16 is out, and it's a big one! In addition to a new website design, we are releasing the first of many features to come allowing you to develop extensions to CloudSharper itself. In this first batch, we are focusing on custom editors for a designated file type.

## Custom editors

[![](http://i.imgur.com/djWYjFP.png)](http://i.imgur.com/LLuR4Mu.png)

Here is how CloudSharper customization works:

 * Every user has a special workspace, called Customizations Workspace, in which you can develop and store your CloudSharper extensions. To open it, click the "Open your customizations workspace" button on your dashboard.

 * Extensions take the form of assemblies located in designated subfolders of this workspace. For example, custom editors will be located in the "Editors" directory.

To make your first custom editor:

* Create a project in your Customizations Workspace by selecting "CloudSharper Editor" in the New Project window. This project contains a simple editor for `.foo` files that simply displays a textarea and responds to Save requests from the UI.

* Build the solution (Ctrl+B). The resulting assembly is automatically copied into the "Editors" folder.

* Select "Reload local custom editors" from the Tools menu to activate the newly built editor in your current session. (existing editors are activated on startup)

* Voilà! You can create a file with extension `.foo` to see your custom editor in action.

Note that if you upload your Customizations Workspace to the cloud, then your extensions will be activated on all machines you use!

## Bugfixes

Here is the bugfix change log for CloudSharper 0.9.16:

 * [#480](https://bitbucket.org/IntelliFactory/cloudsharper/issue/480/closing-tab-doesnt-check-dirty-status): Closing tab doesn't check dirty status
 * [#482](https://bitbucket.org/IntelliFactory/cloudsharper/issue/482/deployed-apps-use-the-wrong-domain-in): Deployed apps use the wrong domain in their URL
 * [#485](https://bitbucket.org/IntelliFactory/cloudsharper/issue/485/getting-the-documentation-page-you): Getting "The documentation page you requested does not exist" on page refresh
 * [#487](https://bitbucket.org/IntelliFactory/cloudsharper/issue/487/add-a-doc-url-to-link-directly-to-a): Add a /doc url to link directly to a documentation page inside the IDE
 * [#489](https://bitbucket.org/IntelliFactory/cloudsharper/issue/489/syncing-down-meta-workspace-fails-to): Syncing down meta workspace fails to create missing folders

Happy coding!
