# Deployment

## Publish

Single-file, self-contained, framework-dependent is the default recommendation — smaller output, still no framework install needed when `SelfContained=true`:

```cmd
dotnet publish -c Release -r win-x64 --self-contained true ^
    /p:PublishSingleFile=true ^
    /p:IncludeNativeLibrariesForSelfExtract=true
```

Result: one `.exe` (~70-100MB depending on dependencies) plus `appsettings.json` and any config files. Drop into a deployment folder, done.

**Don't ship `.pdb` files to production** unless you specifically want stack-trace line numbers in Event Log. They leak implementation details and grow the payload.

## Install scripts

Keep an install script in the repo. Don't rely on operators remembering `sc.exe` syntax.

`install.cmd`:

```cmd
@echo off
setlocal

set SERVICE_NAME=MyCompany.MyService
set DISPLAY_NAME=My Company Service
set BIN_PATH=C:\Services\%SERVICE_NAME%\%SERVICE_NAME%.exe
set SERVICE_ACCOUNT=NT AUTHORITY\NetworkService

sc.exe create "%SERVICE_NAME%" ^
    binPath= "%BIN_PATH%" ^
    start= auto ^
    DisplayName= "%DISPLAY_NAME%" ^
    obj= "%SERVICE_ACCOUNT%"

if %ERRORLEVEL% neq 0 (
    echo Install failed
    exit /b 1
)

sc.exe description "%SERVICE_NAME%" "Long description visible in Services.msc."
sc.exe failure "%SERVICE_NAME%" reset= 3600 actions= restart/5000/restart/30000/restart/60000

echo Install complete. Start with: sc.exe start "%SERVICE_NAME%"
```

PowerShell equivalent is fine if your environment prefers it. Use `New-Service` or `sc.exe` — `New-Service` is less powerful (no recovery config), so `sc.exe` wrapped in PS wins.

## Recovery actions

Every Windows Service should have recovery configured. Defaults are "Take No Action" which is almost never what you want.

```cmd
sc.exe failure "MyCompany.MyService" reset= 3600 actions= restart/5000/restart/30000/restart/60000
```

- `reset= 3600` — reset failure count after 1 hour of clean running.
- `actions=` — first failure: restart after 5s. Second: 30s. Third: 60s.

**Don't configure infinite restart.** A broken service that restarts forever consumes disk space (logs) and hides the fact that it's broken. Three restarts then leave it stopped, and alert on stopped services through your monitoring.

## Service account

### Options ranked

1. **Group Managed Service Account (gMSA)** — best for domain-joined servers. AD rotates the password. Service account has an identity other services can grant permissions to.
2. **Virtual account** (`NT SERVICE\MyServiceName`) — per-service identity, auto-created by SCM. Good for single-machine services.
3. **`NT AUTHORITY\NetworkService`** — built-in, limited privileges, can authenticate to network as the machine account.
4. **`NT AUTHORITY\LocalService`** — like NetworkService but can't access network.
5. **`LocalSystem`** — full local privileges. Default if you don't specify `obj=`. Avoid in production.
6. **Named user account** — personal or shared user. Never. Passwords rotate, accounts get disabled, shared accounts violate least privilege.

### Granting permissions to the service account

After install, grant whatever permissions the service actually needs:

- File/folder access: `icacls` or the ACL UI.
- Registry: `regini` or the registry permissions UI.
- Database: grant to the Windows identity (SQL Server) or the machine account (for gMSA).
- Event Log source: created by installer or first admin run.

Don't grant "full control" anywhere "to make it work." Grant the minimum and log what you granted.

## MSI installer

For shipped products, a proper MSI beats install scripts. Two reasonable tools:

- **WiX v5** — free, XML-based, authoritative. Longer learning curve.
- **Advanced Installer** — commercial, GUI-driven, handles Worker Service registration natively. Faster to get started.

For internal services, scripts are fine. For anything that customers install, use an MSI — it integrates with Windows Add/Remove Programs, supports upgrade, and handles rollback on failure.

## Update procedure

```cmd
sc.exe stop "MyCompany.MyService"
:: wait for STOP_PENDING to clear
timeout /t 5 /nobreak
xcopy /y new-build\*.* C:\Services\MyCompany.MyService\
sc.exe start "MyCompany.MyService"
```

The exe is locked while the service runs. `xcopy` fails silently if you forget the stop. Script this, don't hand-copy.

For zero-downtime: not possible with a single Windows Service instance. Run two services on different ports and load-balance, or accept the few-second restart window.

## CI/CD notes

- Build with `dotnet publish` as shown.
- Sign the exe with a code signing cert if shipping externally (`signtool sign /f cert.pfx ...`).
- Don't check in `appsettings.Production.json` with secrets. Transform at deploy time or use environment variables.
- Tag the build with the git SHA (`AssemblyInformationalVersion`) so logs can correlate to source.

```xml
<PropertyGroup>
  <SourceRevisionId Condition="'$(SourceRevisionId)' == ''">local</SourceRevisionId>
  <InformationalVersion>$(Version)+$(SourceRevisionId)</InformationalVersion>
</PropertyGroup>
```

Log the version on startup:

```csharp
var version = typeof(Program).Assembly
    .GetCustomAttribute<AssemblyInformationalVersionAttribute>()?.InformationalVersion
    ?? "unknown";
logger.LogInformation("Starting MyService version {Version}", version);
```

When operators ask "what's running in prod," the first Event Log entry answers.
