# Installation

## Requirements

- .NET 9.0 or later
- A valid SSL certificate (for production WSS/HTTPS servers)

---

## Install via NuGet CLI

```bash
dotnet add package LuciferCore
```

## Install via Package Manager Console

```powershell
Install-Package LuciferCore
```

## Install via .csproj

```xml
<PackageReference Include="LuciferCore" Version="*" />
```

---

## Verify Installation

After installing, confirm the package is referenced correctly:

```bash
dotnet restore
dotnet build
```

---

## Next Step

Once installed, head to [Quick Start](quick-start.md) to have your first server running in minutes.
