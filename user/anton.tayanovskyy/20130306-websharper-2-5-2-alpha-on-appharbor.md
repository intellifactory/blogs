---
title: "WebSharper 2.5.2-alpha on AppHarbor"
categories: "f#,websharper,appharbor"
abstract: "Pre-release WebSharper 2.5.2-alpha NuGet package is available and can already be used to build AppHarbor-ready sites."
identity: "3054,76218"
---
Pre-release [WebSharper](https://websharper.com) 2.5.2-alpha NuGet package is available and can already be used to build [AppHarbor](https://appharbor.com)-ready sites. AppHarbor is an attractive execution environment for WebSharper apps: its provides a free basic option, and builds and executes your code in the cloud (using Amazon EC2).  The basic option does not come with default permanent storage, but we found that combining it with [Windows Azure](https://windowsazure.com) storage service is entirely viable. Small projects will incur 0 monthly cost because of Azure free storage quota.

Up until now, running WebSharper on AppHarbor without committing the binaries has been a bit quirky. This is finally resolved, and here is a sample ready-to-clone application:
[websharper-bootstrap-site](https://bitbucket.org/IntelliFactory/websharper-bootstrap-site).  We are now finalizing testing the 2.5 WebSharper release and working on preparing a matching extensions release.  In the meanwhile, you can already try to start your own cloud-based WebSharper site with AppHarbor and the alpha package.

Your comments and suggestions on how to improve the app are very welcome.
