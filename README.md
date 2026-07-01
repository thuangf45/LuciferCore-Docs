# LuciferCore

[![NuGet](https://img.shields.io/nuget/v/LuciferCore.svg)](https://www.nuget.org/packages/LuciferCore/)
[![Downloads](https://img.shields.io/nuget/dt/LuciferCore.svg)](https://www.nuget.org/packages/LuciferCore/)

**LuciferCore** is a high-performance **event-driven framework** for modern **modulith** architecture.

It helps you build clean modules in one app today, then split them into microservices later if needed.

LuciferCore combines:
- **OOP** for clear structure
- **DOD (Data-Oriented Design)** for speed and memory efficiency

It uses a **Buffer-Model architecture** to improve cache locality and reduce allocations.

> 🛒 **Commercial license:**  
> If you use LuciferCore in closed-source products, SaaS, or enterprise systems, buy a commercial license at [Lemon Squeezy Store](https://bufmod.lemonsqueezy.com/).  
> This avoids AGPL-3.0 copyleft requirements.

---

## Why LuciferCore?

Most .NET networking frameworks require a lot of manual setup.  
LuciferCore keeps it simple:

```text
Decorate → Implement → Run
```

You write business logic. The framework wires the rest.

---

## Core Concepts

### 1) Buffer-Model Architecture (DOD)

LuciferCore stores and processes data in a DOD-friendly way.

It uses:
- `ObjectPool`
- `ArrayPool`
- SIMD where useful

Benefits:
- Very low allocations
- High throughput
- Lower memory usage
- Better CPU cache locality

### 2) Event-Driven Networking (NetCoreServer)

The network layer is built on **NetCoreServer** with async events.

Benefits:
- Fast event pipeline
- Good concurrency
- Stable memory usage under load

### 3) Modulith + DLL Extension

You can extend features by adding DLL modules.

Common attributes:
- `[Server]`
- `[Manager]`
- `[Middleware]`
- `[Handler]`
- `[Route]`

Modules register themselves, and the core runtime connects them.  
No extra worker-scheduling layer is required.

---

## Install

```bash
dotnet add package LuciferCore
```

---

## Author

Built and maintained by **Nguyen Minh Thuan (thuangf45)**.

- GitHub: [thuangf45](https://github.com/thuangf45)
- NuGet: [thuangf45](https://www.nuget.org/profiles/thuangf45)
- Portfolio: [thuangf45.github.io](https://thuangf45.github.io)
- LinkedIn: [thuangf45](https://www.linkedin.com/in/thuangf45)
- Blog: [dev.to/thuangf45](https://dev.to/thuangf45)
- Email: [kingnemacc@gmail.com](mailto:kingnemacc@gmail.com)

> Building fast .NET systems with practical architecture.