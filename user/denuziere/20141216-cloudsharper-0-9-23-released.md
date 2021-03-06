---
title: "CloudSharper 0.9.23 released"
categories: "cloudsharper,f#,websharper"
abstract: "CloudSharper updates to WebSharper 3.0 alpha."
identity: "4148,77596"
---
It's been a few weeks since our previous release, and finally CloudSharper 0.9.23 is here.

The highlight of this release is the update to WebSharper 3.0.3 alpha. This update concerns:

 * The cloudsharper.com interface itself;
 * The CloudSharper Local Services component;
 * The project templates available inside CloudSharper.

![](http://i.imgur.com/4eQ9KFB.png)

Of course, it is still possible to work on WebSharper 2.5 projects within CloudSharper. Note however that if you want to run a 2.5 sitelets application ("Client-Server WebApplication") from inside CloudSharper, you need to add the following to your web project's `Web.config` file:

```xml
  <runtime>
    <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
       <dependentAssembly>
        <assemblyIdentity name="IntelliFactory.WebSharper.Sitelets"
                          publicKeyToken="dcd983dec8f76a71" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-3.0.0.0" newVersion="2.5.0.0" />
      </dependentAssembly>
      <dependentAssembly>
        <assemblyIdentity name="IntelliFactory.WebSharper.Core"
                          publicKeyToken="dcd983dec8f76a71" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-3.0.0.0" newVersion="2.5.0.0" />
      </dependentAssembly>
    </assemblyBinding>
  </runtime>
```

Happy coding!
