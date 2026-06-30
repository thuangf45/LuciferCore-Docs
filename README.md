# LuciferCore

[![NuGet](https://img.shields.io/nuget/v/LuciferCore.svg)](https://www.nuget.org/packages/LuciferCore/)
[![Downloads](https://img.shields.io/nuget/dt/LuciferCore.svg)](https://www.nuget.org/packages/LuciferCore/)

**LuciferCore** is an all-in-one, high-performance **Event-Driven Ecosystem** engineered explicitly for modern **Modulithic architectures**—allowing you to build clean, monolithic business modules that can easily decouple into high-speed microservices later. Designed by **thuangf45**, it flawlessly bridges the structural elegance of **Object-Oriented Programming (OOP)** with the brutal execution efficiency of **Data-Oriented Design (DOD)**, leveraging a revolutionary **Buffer-Model architecture** to push .NET performance to pure hardware limits while guaranteeing maximum CPU Cache Locality and zero-allocation execution.

> 🛒 **Commercial Licensing:** To integrate LuciferCore into closed-source commercial applications, SaaS platforms, or enterprise infrastructure without AGPL-3.0 copyleft restrictions, please acquire a premium license via our [Lemon Squeezy Store](https://bufmod.lemonsqueezy.com/).

---

## Why LuciferCore?

Most .NET networking frameworks make you a **plumber** — constantly wiring up boilerplate. LuciferCore makes you an **architect**. You decorate, implement, and run. The framework handles everything else.

```
Decorate → Implement → Run
```

---

## Core Pillars

### Buffer-Model Architecture (DOD)

Data is managed through a **Buffer-Model architecture** built on Data-Oriented Design principles. By combining advanced **ObjectPool** and **ArrayPool** strategies with **SIMD**-accelerated processing, LuciferCore achieves near **zero-allocation** execution, blazing-fast throughput, minimal RAM footprint, and excellent **cache locality** — keeping your hot data close to the CPU at all times.

### Event-Driven Networking on NetCoreServer

The networking layer is rebuilt on top of **NetCoreServer**, harnessing .NET's native **async network events** combined with a DOD-first execution model. The result is an event-driven pipeline that's extremely fast, highly concurrent, and resistant to RAM bloat under load.

### Modulithic, DLL-Extensible System

The application architecture follows a **Modulith** design, letting you extend functionality simply by dropping in new DLLs. Modules register independently through attributes like `[Server]`, `[Manager]`, `[Middleware]`, `[Handler]`, and `[Route]`, all wired together by the performance core — with no intermediate worker scheduling layer. Contracts are easy to inherit and extend, giving you near-unmatched flexibility and scalability while keeping performance high. You just focus on writing your business logic.

---

## Quick Install

```bash
dotnet add package LuciferCore
```

---

## Author

Built and maintained by **Nguyen Minh Thuan (thuangf45)**.

- **GitHub:** [thuangf45](https://github.com/thuangf45)
- **NuGet:** [thuangf45](https://www.nuget.org/profiles/thuangf45)
- **Portfolio:** [thuangf45.github.io](https://thuangf45.github.io)
- **LinkedIn:** [thuangf45](https://www.linkedin.com/in/thuangf45)
- **Blog:** [dev.to/thuangf45](https://dev.to/thuangf45)
- **Email:** [kingnemacc@gmail.com](mailto:kingnemacc@gmail.com)

> *Pushing the boundaries of .NET performance, one buffer at a time.*

---
