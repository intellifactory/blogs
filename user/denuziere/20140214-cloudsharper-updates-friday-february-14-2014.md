---
title: "CloudSharper Updates - Friday, February 14, 2014"
categories: "cloudsharper,f#"
abstract: "One day since the first alpha release, and already the first bug reports -- and fixes."
identity: "3729,77056"
---
It's barely been one day since we made CloudSharper Local alpha publicly available, and user feedback is already pouring in. Thanks to everyone who is trying it out, and reporting issues :)

Today we have fixed the following issues:

 * [#259](https://bitbucket.org/IntelliFactory/cloudsharper/issue/259/error-in-console-for-new-user-first-time): If the user hasn't created a workspace yet, a `DirectoryNotFoundException` is logged in the console when trying to list the workspaces.

 * [#262](https://bitbucket.org/IntelliFactory/cloudsharper/issue/262/failing-to-download-template): When running CloudSharper.exe directly, it tries to download templates from localhost:10101 and fails.

CloudSharper alpha is now in version **0.9.4**.

Stay in touch for upcoming updates and features to CloudSharper Local alpha, and don't hesitate to send us feedback on your experience using it!
