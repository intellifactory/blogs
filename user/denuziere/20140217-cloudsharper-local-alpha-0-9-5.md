---
title: "CloudSharper Local alpha 0.9.5"
categories: "cloudsharper,f#"
abstract: "Today's update features an enhanced installer, smoother Markdown and HTML editing, and some more bug fixes."
identity: "3731,77057"
---
We just released CloudSharper Local alpha 0.9.5, with the following changes:

 * [#258](https://bitbucket.org/IntelliFactory/cloudsharper/issue/258/new-project-fails-to-create-a-project-but): When the "New Project" dialog fails to download a template (generally because of a misconfiguration in CloudSharper.Console.exe.config), it warns the user instead of failing silently.

 * [#263](https://bitbucket.org/IntelliFactory/cloudsharper/issue/263/installer-exiting): The installer now shows a friendlier interface, with a final dialog proposing the user to immediately start CloudSharper Local.
 
 * [#264](https://bitbucket.org/IntelliFactory/cloudsharper/issue/264/automatically-refresh-tabbed-document-on): The Markdown and HTML editors feature a preview panel which can be either tabbed or viewed side-by-side next to the source. Until now, the only way to refresh this preview was to click the "Refresh" button. The preview is now automatically refreshed as you type for Markdown, and when the file is saved for HTML.
 
 * The top-right menu now features a link to [this page](http://cloudsharper.com/home), in order to be able to easily download the latest CloudSharper Local when already logged in.

Happy coding!
