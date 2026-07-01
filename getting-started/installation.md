# Installation

## Requirements

- .NET 9.0 or later
- SSL certificate (required only for production WSS/HTTPS)

---

## 1) Install with .NET CLI

```bash
dotnet add package LuciferCore
```

## 2) Install with Package Manager Console

```powershell
Install-Package LuciferCore
```

## 3) Install in `.csproj`

```xml
<PackageReference Include="LuciferCore" Version="*" />
```

---

## Verify

Run:

```bash
dotnet restore
dotnet build
```

If build succeeds, LuciferCore is installed correctly.