---
title: "Making Booster compilation service cross-platform"
categories: "websharper,fsharp,compilation,cross-platform,linux"
abstract: "Change a console application used as a service to support cross-platform. RIDs. What they are good for. Executables across platforms. Detached child process. NamedPipe in linux. IO behavior when using nohup."
identity: "-1,-1"
---

## Intro

Make use of experience from making WebSharper Booster console application (used as service) cross-platform. From Windows only to Linux compatible.

## RIDs

RIDs ([Runtime Identifier][runtime-identifier]) are specifiers for compiler to output operating system, version, architecture, "distro kind" specific code.

That is more information that can be inferred from `RuntimeInformation.IsOSPlatform(OSPlatform.Windows)` or `RuntimeInformation.IsOSPlatform(OSPlatform.Linux)`. For further information check [Platform compatibility analyzer][platform-compatibility-analyzer]. For example it's a valid RID to use `linux-musl-x64`, but the `musl` part is not really version specific. It's not easy to infer it from runtime.

## Why specify RID?

Deploying the application itself doesn't need to be RID qualified ([.NET application publishing overview][dotnet-application-publishing-overview]). Simple `dotnet publish` produces the right `.dll` (cross-platform) and an executable, which is current platform specific.

> Publishing an app as framework-dependent produces a cross-platform binary as a dll file, and a platform-specific executable that targets your current platform

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

Alternative would be using the cross-platform `.dll` with `dotnet wsfscservice.dll` but this makes hard to find the running process through all running `dotnet` instances. [Rename the running process is not possible][rename-the-running-process-is-not-possible] or require `start "wsfscservice" dotnet wsfscservice.dll` "hack" which is Windows specific. Also finding the process programatically is simpler:

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

## `<RuntimeIdentifier>` vs `<RuntimeIdentifiers>`

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

It was a surprise that `--self-contained`'s default value is variable. If you specify RID, it's `true` if you just leave `dotnet publish` it's `false`. So to avoid [Publish self-contained][publish-self-contained], `--self-contained false` had to be added.

[publish-self-contained]: <https://docs.microsoft.com/en-us/dotnet/core/deploying/#publish-self-contained>
[framework-dependent]: <https://docs.microsoft.com/en-us/dotnet/core/deploying/#publish-framework-dependent>
[reattach-debugger]: <https://github.com/dotnet/runtime/issues/2688#issuecomment-370030584>
[rename-the-running-process-is-not-possible]: <https://github.com/dotnet/runtime/issues/2688>
[dotnet-application-publishing-overview]: <https://docs.microsoft.com/en-us/dotnet/core/deploying/>
[platform-compatibility-analyzer]: <https://docs.microsoft.com/en-us/dotnet/standard/analyzers/platform-compat-analyzer>
[runtime-identifier]: <https://docs.microsoft.com/en-us/dotnet/core/rid-catalog>