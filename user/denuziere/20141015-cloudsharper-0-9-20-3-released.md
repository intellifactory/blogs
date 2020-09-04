---
title: "CloudSharper 0.9.20.3 released"
categories: "cloudsharper,f#,websharper"
abstract: "This release of CloudSharper adds configuration options for the local service's web server and the workspaces directory."
identity: "4064,77473"
---
This release of CloudSharper adds configuration options for the local service's web server and the workspaces directory. These options are present in the `CloudSharper.Console.exe.config` file if you use the GUI console, or can be passed via the command line if you use `CloudSharper.exe` directly.

 * [#558](https://bitbucket.org/IntelliFactory/cloudsharper/issue/558/make-root-folder-path-configurable): Added an option `rootdir` pointing to the directory where CloudSharper should store workspace files.

    The string `"$USERPROFILE"` is replaced with the user's home directory (usually `C:\Users\username` in Windows).

    The default value is `"$USERPROFILE\CloudSharper"`.

 * [#559](https://bitbucket.org/IntelliFactory/cloudsharper/issue/559/run-the-web-servers-on-localhost-instead): Removed the option `serveip` and replaced it with two options: `websocketserverip` and `webserverhostname`.

    * `websocketserverip` configures the IP address on which the WebSockets server listens.

        The default is `127.0.0.1`.

    * `webserverhostname` configure the hostname or IP address on which the web servers (for static files and WebSharper Sitelets) listen.

        The default is `localhost`.


### Note for users in restricted environments such as Windows Domain accounts

CloudSharper used to require the use of the `netsh` command with administrator privilege in order to be able to run its web servers. Starting with this version 0.9.20.3, this is not the case anymore when `webserverhostname="localhost"`. However, you may still experience a failure to start the web server and an error message printed by CloudSharper advising you to use `netsh` to allow the server to run. The most likely reason is that the port was reserved for previous CloudSharper versions to serve under the hostname `127.0.0.1` instead of `localhost`. The best course of action is to remove this reservation by running the following command as administrator:

```
netsh http delete urlacl http://127.0.0.1:PORT/
```

where `PORT` is the port number indicated in the error message.

You do *not* need to then run the `netsh http add` command as advised in the error message, because `localhost` is allowed by default if no other hostname reserves the same port.

If you have any questions, don't hesitate to [ask us on the FPish forums](http://fpish.net/new-topic/1/cloudsharper).

Happy coding!
