---
title: "Chocolatey"
categories: "fake,automation,chocolatey"
abstract: "Chocolatey looks very useful - it lets you install software into PATH on Windows by one-liner commands from PowerShell. A poor-man's package manager."
identity: "3313,76533"
---
If you have not yet, check [Chocolatey](https://chocolatey.org/) out. It looks very useful. This little utility lets you install software into PATH on Windows by one-liner commands from PowerShell. It is a poor-man's package manager for Windows.

In particular, it automatically knows about NuGet packages. Packages with executable files under "tools" get aliased into `C:\Chocolatey\Bin` or similar, and the tools end up on your PATH.

For example, to get started with [FAKE](http://github.com/fsharp/fake) - the F# Make you can do:

```fsharp
cinst -pre FAKE
mkdir my-project
cd my-project
fake boot init one # initializes the project, creates build.fsx
./build.fsx # edit your build script in VS
fake # build
```
