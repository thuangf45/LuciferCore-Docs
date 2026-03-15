# LuciferCore

[![NuGet](https://img.shields.io/nuget/v/LuciferCore.svg)](https://www.nuget.org/packages/LuciferCore/)
[![Downloads](https://img.shields.io/nuget/dt/LuciferCore.svg)](https://www.nuget.org/packages/LuciferCore/)

**LuciferCore** is an all-in-one, high-performance **Event-Driven Ecosystem** for .NET, built on the principles of **Data-Oriented Design (DOD)**. It leverages a revolutionary **Buffer-Model architecture** to push .NET performance to hardware limits — ensuring maximum CPU Cache Locality and zero-allocation execution.

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

High-speed dispatching via `[Server]`, `[Handler]`, `[WsMessage]`, and HTTP verb attributes — no manual routing tables, no reflection overhead at runtime.

### Built-in Security

Native defense mechanisms with `[RateLimiter]`, `[Authorize]`, and `[Safe]` — applied directly at the handler level.

### Hybrid Protocols

Handle **WSS (WebSocket Secure)** and **HTTPS** within the same unified logic and server instance.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Server / Session | `NetCoreServer` principles |
| Manager Loop | Unity Loop architecture |
| Async Dispatch | `System.Threading.Channels` |
| Memory | `Span<T>` virtualization + Object Pooling |
| Memory Copy | SIMD-accelerated |
| Concurrency | Lock-free queues |

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

Licensed under the **MIT License**.
