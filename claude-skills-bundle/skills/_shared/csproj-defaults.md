# csproj defaults — .NET 10 / C# 14

Common properties used across the C# skills (`csharp-avalonia`, `csharp-rest-aspnetcore`, `csharp-efcore-sqlite`, `csharp-windows-service`, `csharp-sensor-pipeline`). Each skill repeats them in its own csproj snippet so the snippet stays copy-pasteable; this file is the canonical explanation of *why*.

## The common block

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

## Why each property

### `TargetFramework=net10.0`

Default for cross-platform code. Use `net10.0-windows` only when a Windows-specific package or API is involved — currently just the `csharp-windows-service` skill, where `Microsoft.Extensions.Hosting.WindowsServices` pulls in Windows interop. Avalonia is cross-platform; despite running on Windows, it stays on plain `net10.0` unless you call Win32 APIs directly.

### `Nullable=enable`

Reference-type nullability is part of the type system. The compiler tracks which references can be null and forces you to handle them. Catches null bugs at build time rather than as `NullReferenceException` in production. Non-negotiable for new code.

### `ImplicitUsings=enable`

Removes the boilerplate `using` block at the top of every file. The implicit set is SDK-specific — `Microsoft.NET.Sdk`, `Microsoft.NET.Sdk.Web`, and `Microsoft.NET.Sdk.Worker` each include the right namespaces for their workload. Trust the SDK; don't repeat the standard usings manually.

### `TreatWarningsAsErrors=true`

Compiler warnings become build failures. Forces the team to fix them in the moment, not in a "warnings cleanup" sprint that never lands. If a specific warning is genuinely benign in context, suppress it locally with `#pragma warning disable CS####` plus a one-line comment explaining why — don't lower the global bar.

## SDK-specific additions

Each skill layers its own properties on top of the common block:

| Skill | Adds |
|---|---|
| `csharp-avalonia` | `OutputType=WinExe`, `BuiltInComInteropSupport`, `ApplicationManifest`, `AvaloniaUseCompiledBindingsByDefault` |
| `csharp-rest-aspnetcore` | `InvariantGlobalization`, `InterceptorsNamespaces` (Minimal API validation source generator) |
| `csharp-windows-service` | overrides `TargetFramework` to `net10.0-windows`; adds `OutputType=exe`, `RuntimeIdentifier=win-x64`, `PublishSingleFile`, `SelfContained` |
| `csharp-efcore-sqlite` | nothing beyond defaults; the EF Core packages do the work |
| `csharp-sensor-pipeline` | inherits from its host (Worker SDK or Web SDK) |

See each skill's `SKILL.md` for the full csproj including its package references and the rationale for any SDK-specific additions.
