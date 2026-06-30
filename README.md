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

### Zero-Allocation Buffer-Model

LuciferCore treats memory as a contiguous, virtualized pool. Data flows from raw socket bytes to your handler through a pre-pooled, zero-copy pathway — virtually eliminating GC pressure.

### Data-Oriented Design (DOD)

By focusing on data streams rather than object hierarchies, LuciferCore minimizes pointer chasing and memory fragmentation. Your data is almost always hot in **L1/L2/L3 Cache**.

### Hardware-Scale Parallelism

Built on fully asynchronous, lock-free pathways using **Async Channels** and **Lock-free Queues**. The engine saturates all available CPU cores without contention or context-switching.

### Attribute-Driven Routing

High-speed dispatching via `[Server]`, `[Handler]`, `[WsMessage]`, `[Middleware]` and HTTP verb attributes — no manual routing tables, no reflection overhead at runtime.

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
