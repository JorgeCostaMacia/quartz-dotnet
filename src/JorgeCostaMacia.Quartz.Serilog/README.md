# JorgeCostaMacia.Quartz.Serilog

**Serilog observability for Quartz**: job and trigger log listeners that emit **fixed, low-cardinality messages** (`JobToBeExecuted`, `JobWasExecuted`, `TriggerFired`, `TriggerMisfired`…) and push everything variable — scheduler, job, trigger, data, times, refire/retry counters and the trace identifiers — through the Serilog log context. Cheap to index (Loki/Grafana-friendly), structural to search.

[![NuGet](https://img.shields.io/nuget/v/JorgeCostaMacia.Quartz.Serilog.svg)](https://www.nuget.org/packages/JorgeCostaMacia.Quartz.Serilog/)
[![Downloads](https://img.shields.io/nuget/dt/JorgeCostaMacia.Quartz.Serilog.svg)](https://www.nuget.org/packages/JorgeCostaMacia.Quartz.Serilog/)
[![Build](https://github.com/JorgeCostaMacia/quartz-dotnet/actions/workflows/main.yml/badge.svg?branch=main)](https://github.com/JorgeCostaMacia/quartz-dotnet/actions/workflows/main.yml)
[![License](https://img.shields.io/github/license/JorgeCostaMacia/quartz-dotnet.svg)](https://github.com/JorgeCostaMacia/quartz-dotnet/blob/main/LICENSE.txt)

---

## Install

```bash
dotnet add package JorgeCostaMacia.Quartz.Serilog
```

## Usage

```csharp
using JorgeCostaMacia.Quartz.Serilog;
using JorgeCostaMacia.Quartz.Serilog.Infrastructure;

services
    .AddSerilogContext()   // registers both listeners as singletons
    .AddQuartz(q =>
    {
        // ... your scheduler/store configuration ...
        q.AddJobListener<JobsLoggerListener>();
        q.AddTriggerListener<TriggerLoggerListener>();
    });
```

- **`JobsLoggerListener`** — `JobToBeExecuted` (Information), `JobExecutionVetoed` (Warning), `JobWasExecuted` (Information, or **Error with the root cause attached** when the job threw).
- **`TriggerLoggerListener`** — `TriggerFired` / `TriggerComplete` (Information, with the scheduler's resulting instruction), `VetoJobExecution` (Information, never vetoes), `TriggerMisfired` (**Error** — the trigger missed its scheduled time), `TriggerRetriesExhausted` (**Warning, with the root cause attached** — a trigger with a retry policy gave up on a failed firing; the failure itself was already logged by `JobWasExecuted`).
- Every event carries `AggregateId`/`CorrelationId` from [`JobTrace`](https://www.nuget.org/packages/JorgeCostaMacia.Quartz/) — idempotent get-or-create on the firing's own data map, so your own listeners (e.g. event publishers) read the same pair with `JobTrace.GetOrCreate(context)`, in any order.

## Requirements

The **.NET 10** SDK.

Depends on [JorgeCostaMacia.Quartz](https://www.nuget.org/packages/JorgeCostaMacia.Quartz/), [Quartz](https://www.nuget.org/packages/Quartz/), [Serilog](https://www.nuget.org/packages/Serilog/) and the `Microsoft.Extensions` logging and dependency-injection abstractions.

## About

`JorgeCostaMacia.Quartz.Serilog` is part of **[quartz-dotnet](https://github.com/JorgeCostaMacia/quartz-dotnet)** — extensions for Quartz.NET: job-execution trace correlation, the clustered Postgres store and its defaults, and Serilog job and trigger listeners, each package scoped to a single concern. An independent project, not affiliated with Quartz.NET or Serilog: these listeners extend Quartz through its listener interfaces and write through Serilog's public API.

- **Repository:** [github.com/JorgeCostaMacia/quartz-dotnet](https://github.com/JorgeCostaMacia/quartz-dotnet)
- **Issues & requests:** [open an issue](https://github.com/JorgeCostaMacia/quartz-dotnet/issues)
- **Contributing:** [CONTRIBUTING.md](https://github.com/JorgeCostaMacia/quartz-dotnet/blob/main/CONTRIBUTING.md)
- **Security:** [report a vulnerability](https://github.com/JorgeCostaMacia/quartz-dotnet/security/advisories/new)

**Author:** Jorge Costa Maciá

- [LinkedIn](https://www.linkedin.com/in/jorge-costa-macia-842817164/)
- [GitHub](https://github.com/JorgeCostaMacia/)
- [Bitbucket](https://bitbucket.org/jorgecostamacia/)
