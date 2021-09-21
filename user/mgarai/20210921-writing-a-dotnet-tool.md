---
title: "Writing a dotnet tool"
categories: "dotnet,tool,cli,fsharp"
abstract: "Writing a dotnet tool with help of the dotnet CLI"
identity: "-1,-1"
---

## Intro

Writing a dotnet tool from scratch have a lot of support from the dotnet CLI. With the following simple steps you can have a tool from the dotnet infrastructure.

## Steps

Create the console application for the tool:
```ps1
> dotnet new console -lang F# -n <tool-name>
> cd <tool-name>
```

For the `tool-name` check the next section

(Optional) Add a solution file for the project:
```ps1
> dotnet new sln
> dotnet sln add <tool-name>.fsproj
```

Check build
```ps1
> dotnet build
```

Edit Program.fs. A dotnet tool is basically a console application, so implement what you like.

Add necessary variables for the `.fsproj`. You should see something like this:
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net5.0</TargetFramework>
    <RootNamespace><tool_name></RootNamespace>
    <WarnOn>3390;$(WarnOn)</WarnOn>
    <!-- these 2 are the added variables -->
    <PackAsTool>true</PackAsTool>
    <ToolCommandName><tool-name></ToolCommandName>
  </PropertyGroup>

  <ItemGroup>
    <Compile Include="Program.fs" />
  </ItemGroup>

</Project>
```

Add `tool-manifest`:
```ps1
> dotnet new tool-manifest
```

Edit `.config\dotnet-tools.json` (the tool manifest). Specify the version and available commands like this:
```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "<tool-name>": {
      "version": "1.0.0",
      "commands": [ <available-commands> ]
    }
  }
}
```

Run pack
```ps1
> dotnet pack
> cd bin/Debug
```

Install the new tool globally. Here the `tool-name` is not the `.nupkg` file name. It's just the `<tool-name>` given at `dotnet new -n <tool-name>`.
```ps1
> dotnet tool install -g --add-source . <tool-name>
```

Verify installation:
```ps1
> cd %USERPROFILE%\.dotnet\tools
or
> cd $HOME/.dotnet/tools
---
> dir
or 
> ls
```

You have a dotnet tool installed.

There is also a [tutorial][tutorial] for that in the `Microsoft` documentation.

## Name of the executable. \<ToolCommandName /\>

Before just creating the console application with the come up name for the tool, consider if `dotnet-` prefix should be added or not.

It's important to understand if you issue command
```ps1
> dotnet xy
```
in the dotnet ecosystem it's searching for a tool named `xy` prefixed by `dotnet-` and runs it. If anywhere in your `PATH` variable's directories there is a `dotnet-xy` it will run it. It's just a nice to have to install them where `dotnet install` puts `dotnet-xy`.

The installation is versioned. At the installation destination there will be subdirectories for the installed versions like that:

![](/assets/store-folder.png)

The important question is the usage of the tool. For `dotnet xy` go with `dotnet-xy`. For `xy` go with `xy`.

Choosing between the 2 is first step, so the main directory at `dotnet new console -n xy` can be in par with the final executable's name.

For more information go to [this tutorial][custom-location].

### ToolCommandName

The `<AssemblyName>` variable in the `.csproj`/`.fsproj` file doesn't set the executable's name when installed into tools. `<ToolCommandName>` does. `dotnet pack` will use `<ToolCommandName>` which goes into 
`%USERPROFILE%\.dotnet\tools` or `$HOME/.dotnet/tools` or where `--tool-path` points to in `dotnet tool install --tool-path ...`. All should be in the `PATH` environment variable (be careful, `--tool-path` is not by default).

## FAQ
**Question:** I changed the code, but no change is visible in the installed tool. Why?

**Answer:** Have you repacked the changes?
```ps1
> dotnet pack
> cd bin/Debug
> dotnet tool install -g --add-source . <tool-name>
```

Have you changed version, and forgot to bump it accidentally?
```ps1
> dotnet tool install -g --add-source . --version 1.0.0 <tool-name>
instead of
> dotnet tool install -g --add-source . --version 1.1.0 <tool-name>
```

###
**Q:** I have the following error:
```
error NU1101: Unable to find package <tool-name>. No packages exist with this id in source(s): dotnet-websharper-GitHub, fsc feed, Microsoft Visual Studio Offline Packages, nuget.org
The tool package could not be restored.
Tool '<tool-name>' failed to install. This failure may have been caused by:

* You are attempting to install a preview release and did not use the --version option to specify the version.
* A package by this name was found, but it was not a .NET tool.
* The required NuGet feed cannot be accessed, perhaps because of an Internet connection problem.
* You mistyped the name of the tool.

For more reasons, including package naming enforcement, visit https://aka.ms/failure-installing-tool
```
How can I solve it?

**A:** This can mean anything. Usually the problem is missing `--add-source <folder-to-.nupkg>`, or using the file name instead of `[PACKAGE_NAME]`.
```ps1
> dotnet tool install -g --version 1.0.0 <tool-name>.1.0.0.nupkg
instead of
> dotnet tool install -g --add-source . --version 1.0.0 <tool-name>
```
###
**Q:** Can't find the executable in `%USERPROFILE%\.dotnet\tools`. Why?

**A:** Have you provided `-g` flag?
```ps1
> dotnet tool install -g ...
```
###
**Q:** `dotnet tool list` doesn't list what I have provided in the tools manifest. Why?

**A:** `dotnet tool list` also differentiate between global and local tools. Have you added `-g` flag for your installed global tool?
```ps1
> dotnet tool list -g
```

[tutorial]: <https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools-how-to-create>
[custom-location]: <https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools-how-to-use#use-the-tool-as-a-global-tool-installed-in-a-custom-location>
