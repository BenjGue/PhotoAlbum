---
name: dotnet-upgrade
description: >-
  Transformation rules and verification steps for upgrading the PhotoAlbum
  ASP.NET Core Razor Pages app (EF Core + SQL Server) from .NET 9 to .NET 10
  and making it ready for Azure Container Apps.
  WHEN: upgrade to .NET 10, retarget net9.0 to net10.0, bump EF Core to 10,
  update Dockerfile to .NET 10 images, prepare PhotoAlbum for Azure Container
  Apps, fix CVE in NuGet packages, remove leftover web.config.
  NOT for: .NET Framework 4.x migrations (System.Web, WCF), greenfield .NET
  apps, Java projects, Azure infrastructure provisioning (Bicep/azd).
---

# PhotoAlbum — .NET 9 → .NET 10 Upgrade Rules

## Purpose

Packages the project-file rules, package mappings, cloud-readiness rules, and
verification checklist needed to retarget PhotoAlbum to .NET 10 without behavioral change.

## Prerequisites

- `ASSESSMENT.md` exists and is approved.
- .NET 10 SDK installed — verify: `dotnet --list-sdks` (a `10.0.x` entry is required).
- `dotnet-ef` tool matching EF Core 10 — verify: `dotnet ef --version`
  (update with `dotnet tool update -g dotnet-ef`).

## Procedure

### Step 1 — Retarget projects (web app first, then tests)

```powershell
# Edit <TargetFramework> in both projects, then:
dotnet restore PhotoAlbum.sln
dotnet build PhotoAlbum.sln
```

- `PhotoAlbum/PhotoAlbum.csproj` → `<TargetFramework>net10.0</TargetFramework>`.
- `PhotoAlbum.Tests/PhotoAlbum.Tests.csproj` → `<TargetFramework>net10.0</TargetFramework>`.
- Upgrade every `9.0.x` Microsoft package to the latest `10.0.x` in the **same** commit
  (mixed 9/10 EF Core packages fail at runtime).

### Step 2 — Apply transformation rules

| Source pattern (current) | Target pattern (.NET 10) | Notes |
|---|---|---|
| `<TargetFramework>net9.0</TargetFramework>` | `<TargetFramework>net10.0</TargetFramework>` | Both projects |
| `Microsoft.EntityFrameworkCore.SqlServer` / `.Design` `9.0.9` | `10.0.x` | Keep both on the exact same version |
| `Microsoft.AspNetCore.Mvc.Testing` / `EntityFrameworkCore.InMemory` `9.0.9` | `10.0.x` | Test project |
| `mcr.microsoft.com/dotnet/sdk:9.0` / `aspnet:9.0` in `Dockerfile` | `sdk:10.0` / `aspnet:10.0` | Keep `EXPOSE 8080` (default ASP.NET Core container port) |
| Leftover `PhotoAlbum/Web.config` with `(localdb)` connection string | Delete the file | Ignored by Kestrel; misleading on Linux containers |
| `(localdb)` connection string in `appsettings.json` | Override with env var `ConnectionStrings__DefaultConnection` | Prefer Microsoft Entra auth (`Authentication=Active Directory Default`) on Azure SQL |
| `app.UseHttpsRedirection()` behind the Container Apps ingress | `app.UseForwardedHeaders()` (XForwardedFor + XForwardedProto) **before** it | TLS is terminated at the ingress |
| `Admin:Password` in configuration | Env var `Admin__Password` sourced from a Container Apps secret / Key Vault | Never commit it to `appsettings*.json` |

### Step 3 — Verify

- [ ] `dotnet build PhotoAlbum.sln` zero errors and no new warnings
- [ ] No remaining `net9.0` or `9.0.` Microsoft package references (grep `*.csproj` and `Dockerfile`)
- [ ] `dotnet ef migrations has-pending-model-changes --project PhotoAlbum` reports no changes
- [ ] Tests pass: `dotnet test PhotoAlbum.sln`
- [ ] `docker build .` succeeds and the container answers `GET /` with HTTP 200
- [ ] CVE scan clean of criticals (`appmod-dotnet-cve-check`)

## Common Pitfalls

- ⚠️ **Mixed EF Core versions** — `SqlServer` and `Design` must match, or you get `MissingMethodException` at startup.
- ⚠️ **Local uploads are ephemeral** — `wwwroot/uploads` is lost on every Container Apps revision/restart and not shared between replicas. Plan the move to Azure Blob Storage (`Azure.Storage.Blobs` + `DefaultAzureCredential`) in Challenge 3; `PhotoService` and `PhotoFile.cshtml.cs` both touch the disk.
- ⚠️ **Inconsistent upload root** — `PhotoFile.cshtml.cs` uses `Directory.GetCurrentDirectory()` while `Program.cs` uses `ContentRootPath`; they only match because the container `WORKDIR` is `/app`.
- ⚠️ **Cookie auth across replicas** — without shared Data Protection keys (e.g. `Azure.Extensions.AspNetCore.DataProtection.Blobs`), admin logins break after a restart or when scaling out.
- ⚠️ **`MigrateAsync()` at startup** runs on every replica — with more than one replica, run migrations once (job or deployment step) instead.
- ⚠️ **.NET 10 container images** changed their default Linux distribution — re-check any `apt-get` step added later to the `Dockerfile`.

## References

- Microsoft Learn: "What's new in .NET 10" and "Breaking changes in .NET 10"
- Microsoft Learn: "Breaking changes in EF Core 10"
- Microsoft Learn: "Configure ASP.NET Core to work with proxy servers and load balancers"
- AppCAT rule documentation (linked from `ASSESSMENT.md` findings)
