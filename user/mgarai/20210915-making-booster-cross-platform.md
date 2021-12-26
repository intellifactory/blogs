---
title: "Making Booster compilation service cross-platform"
categories: "websharper,fsharp,compilation,cross-platform,linux"
abstract: "Change a console application used as a service to support cross-platform. RIDs. What they are good for. Executables across platforms. Detached child process. NamedPipe in Linux. IO behavior when using nohup."
identity: "-1,-1"
---

## Intro

Make use of experience from making WebSharper Booster console application (used as service) cross-platform. From Windows only to Linux compatible.

## RIDs

RIDs ([Runtime Identifier][runtime-identifier]) are specifiers for compiler to output operating system, version, architecture, "distro kind" specific code.

That is more information that can be inferred from `RuntimeInformation.IsOSPlatform(OSPlatform.Windows)` or `RuntimeInformation.IsOSPlatform(OSPlatform.Linux)`. For further information check [Platform compatibility analyzer][platform-compatibility-analyzer]. For example it's a valid RID to use `linux-musl-x64`, but the `musl` part is not really version specific. It's not easy to infer it from runtime.

### Why specify RID?

Deploying the application itself doesn't need to be RID qualified ([.NET application publishing overview][dotnet-application-publishing-overview]). Simple `dotnet publish` produces the right `.dll` (cross-platform) and an executable, which is current platform specific.

> Publishing an app as framework-dependent produces a cross-platform binary as a `.dll` file, and a platform-specific executable that targets your current platform

When cross-platform executable is needed, than the `dotnet publish` command needs a `-r` flag. Which is the RID. The `<RuntimeIdentifier>` tag in the project file specifies RID.

If the executable expected to be of the right name:

Windows specific
```shell
wsfscservice.exe
```

Linux specific
```bash
./wsfscservice
```

I need to specify RID. This helps finding the running service from the `Task Manager` or `ps`.

### Current platform's RID?

As mentioned before the RID have more information coded in itself than `RuntimeInformation` namespace gives us. If the execution of the application dispatched through `.targets` file definition, we can use the RID that's available there for .NET Core SDK Runtime Identifier. I found that variable while checking through SDK's code reverse engineering how `_UsingDefaultPlatformTarget` is set. For `_UsingDefaultPlatformTarget` = `true` something is used as [default RID][default-runtime-id].

```xml
<DefaultAppHostRuntimeIdentifier>$(NETCoreSdkRuntimeIdentifier)</DefaultAppHostRuntimeIdentifier>
```

Some [information][netcoresdkruntimeidentifier-information] about the `MSBuild` variable:

> The NETCoreSdkRuntimeIdentifier MSBuild property determines the bitness of *.comhost.dll

This is a good default value to find out the folder where platform specific `.dll` is located:

```xml
<!-- where platformspecific wsfscservice/wsfscservice.exe located -->
<FscToolPath>$(MSBuildThisFileDirectory)/../tools/net5.0/$(NETCoreSdkRuntimeIdentifier)/</FscToolPath>
```

In the two cases tested `$(NETCoreSdkRuntimeIdentifier)` is substituted for `win-x64` and `linux-x64`.

### Keeping the process's name across different platforms

Alternative would be using the cross-platform `.dll` with `dotnet wsfscservice.dll` but this makes hard to find the running process through all running `dotnet` instances. [Rename the running process is not possible][rename-the-running-process-is-not-possible] or require `start "wsfscservice" dotnet wsfscservice.dll` "hack" which is Windows specific. Also finding the process programmatically is simpler:

```fsharp
let runningServers =
    try
        Process.GetProcessesByName("wsfscservice")
        |> Array.filter (fun x -> System.String.Equals(x.MainModule.FileName, fileNameOfService, System.StringComparison.OrdinalIgnoreCase))
    with
    | e ->
        // If the processes cannot be queried, because of insufficient rights, the Mutex in service will handle
        // not running 2 instances
        nLogger.Error(e, "Could not read running processes of wsfscservice.")
        [||]
```

Attach/reattach debugger is [easier][reattach-debugger].

So the choice is [framework-dependent][framework-dependent] but non `--self-contained`. `--self-contained` would make the executable bigger, and makes harder to patch the runtime around the executable.

### `<RuntimeIdentifier>` vs `<RuntimeIdentifiers>`

Expecting that using `<RuntimeIdentifiers>` instead of `<RuntimeIdentifier>` (mind the plural) changes the output structure of published nuget is wrong. When used correctly the RID specific output should be scattered around in `Debug`/`Release` folder (inside for example `win-x64`, `linux-x64`, `linux-musl-x64`). But `dotnet publish` doesn't publish for all different RIDs specified in the `<RuntimeIdentifiers>`. Instead uses directly the `Debug`/`Release` folder. Just like the `dotnet build` command doesn't.

![](/assets/outputs.png)

For that to work we needed RIDs count `publish` command for all executables. Libraries doesn't need extra care. In our build process it looks like this (Fake specific. `DotNet.publish` calls `dotnet publish`)

```fsharp
let publishExe (mode: BuildMode) fw input output =
    for rid in [ "win-x64"; "linux-x64"; "linux-musl-x64" ] do
        let outputPath =
            __SOURCE_DIRECTORY__ </> "build" </> mode.ToString() </> output </> fw </> rid </> "deploy"
        DotNet.publish (fun p ->
            { p with
                Framework = Some fw
                OutputPath = Some outputPath
                NoRestore = true
                SelfContained = false |> Some // add --self-contained false
                Runtime = rid |> Some // add -r <RID> from the list above.
                Configuration = DotNet.BuildConfiguration.fromString (mode.ToString())
            }) input
...
BuildAction.Custom <| fun mode ->
    publishExe mode "net5.0" "src/compiler/WebSharper.FSharp/WebSharper.FSharp.fsproj" "FSharp"
    publishExe mode "net5.0" "src/compiler/WebSharper.FSharp.Service/WebSharper.FSharp.Service.fsproj" "FSharp"
    publishExe mode "net5.0" "src/compiler/WebSharper.CSharp/WebSharper.CSharp.fsproj" "CSharp"
```

This also can be achieved by simple shell script, which iterates on the RIDs specified in the `<RuntimeIdentifiers>`. If the singular `-r` parameter is not from the listed `<RuntimeIdentifiers>` RIDs from the project file the `dotnet publish` will fail!

Important tags to the project file to be added:

```xml
<OutputType>Exe</OutputType>
<TargetFrameworks>net5.0</TargetFrameworks>
<Name>wsfscservice</Name>
<AssemblyName>wsfscservice</AssemblyName>
<RuntimeIdentifiers>win-x64;linux-x64;linux-musl-x64</RuntimeIdentifiers>
```

### Self contained

It was a surprise that `--self-contained`'s default value is variable. If you specify RID, it's `true` if you just leave `dotnet publish` it's `false`. So to avoid [Publish self-contained][publish-self-contained], `--self-contained false` had to be added.


## Running detached child processes

When spinning off an executable the process will become the child process of the main executable. That means however someone starts a process to be background it will be closing with the parent executable. Our Console application can't function as a service.

Starting the executable can be:

```fsharp
// start a detached wsfscservice.exe. Platform specific.
let cmdName = if System.Runtime.InteropServices.RuntimeInformation.IsOSPlatform(System.Runtime.InteropServices.OSPlatform.Windows) then
                "wsfscservice.exe" else "wsfscservice"
let cmdFullPath = (location, cmdName) |> System.IO.Path.Combine
let startInfo = ProcessStartInfo(cmdFullPath)
startInfo.CreateNoWindow <- true
startInfo.UseShellExecute <- false
startInfo.WindowStyle <- ProcessWindowStyle.Hidden
proc <- Process.Start(startInfo)
```

But it will be still child process. The solution is to start the process operation system specifically. Create a `.cmd` for Windows using `start` with `/b` parameter to be [background process][windows-background]. 

```shell
@echo off

start /d %~dp0 /b wsfscservice.exe
```

And a `.sh` for Linux. Here [`nohup`][nohup] makes `wsfscservice` immune to hangups and `&` at the end makes it [background process][linux-background].

> If a command is terminated by the control operator &, the shell executes the command in the background in a subshell. The shell does not wait for the command to finish, and the return status is 0

```bash
#!/usr/bin/env bash

nohup "$(dirname "$BASH_SOURCE")/wsfscservice" >/dev/null 2>&1 &
```

`Process.Start` can start the platform specific `.cmd`/`.sh`. No matter it returned immediately with `0` code it's sure that the background process is running. `Task Manager` shows `wsfscservice` and `ps -al` show `wsfscservice`. "Start new process, without being a child of the spawning process" question on `Stackoverflow.com` top voted answer also does [something similar][so-use-stub-exe].

```fsharp
let cmdName = if System.Runtime.InteropServices.RuntimeInformation.IsOSPlatform(System.Runtime.InteropServices.OSPlatform.Windows) then
                "wsfscservice_start.cmd" else "wsfscservice_start.sh"
```

## Caveat around `nohup`

The service processes commands in an infinite loop. This can be achieved by startup a mailbox processor with `Async.Start`, then on the main thread block the execution. At first blocking the main thread was done by `Console.ReadLine() |> ignore`. Avoiding busy wait. But `nohup`'s IO piping makes this line of code just continue.

Another possible solution for non-busy wait is using:

```fsharp
use locker = new AutoResetEvent(false)
locker.WaitOne() |> ignore
```

That solved that a code working on Windows now keeps the service running on Linux as well.

## NamedPipes on Linux

One thing it's worth mentioning is `NamedPipe`'s `PipeTransmissionMode.Message` is [not supported][not-supported] in the Linux runtime. Had to reimplement the `IsMessageComplete` behavior on `PipeTransmissionMode.Byte` either by pre-sending bytes count (how much byte will a full message be independent of buffer size), or sending separator byte sequence between messages.

[not-supported]: <https://github.com/dotnet/runtime/blob/main/src/libraries/System.IO.Pipes/ref/System.IO.Pipes.cs#L145-L146>
[so-use-stub-exe]: <https://stackoverflow.com/a/8434682/1859959>
[linux-background]: <https://linux.die.net/man/1/bash>
[nohup]: <https://linux.die.net/man/1/nohup>
[windows-background]: <https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/start>
[netcoresdkruntimeidentifier-information]: <https://docs.microsoft.com/en-us/dotnet/core/native-interop/expose-components-to-com>
[default-runtime-id]: <https://github.com/dotnet/sdk/blob/main/src/Tasks/Microsoft.NET.Build.Tasks/targets/Microsoft.NET.RuntimeIdentifierInference.targets>
[publish-self-contained]: <https://docs.microsoft.com/en-us/dotnet/core/deploying/#publish-self-contained>
[framework-dependent]: <https://docs.microsoft.com/en-us/dotnet/core/deploying/#publish-framework-dependent>
[reattach-debugger]: <https://github.com/dotnet/runtime/issues/2688#issuecomment-370030584>
[rename-the-running-process-is-not-possible]: <https://github.com/dotnet/runtime/issues/2688>
[dotnet-application-publishing-overview]: <https://docs.microsoft.com/en-us/dotnet/core/deploying/>
[platform-compatibility-analyzer]: <https://docs.microsoft.com/en-us/dotnet/standard/analyzers/platform-compat-analyzer>
[runtime-identifier]: <https://docs.microsoft.com/en-us/dotnet/core/rid-catalog>
