# quartz-dotnet — working in this repo

Extensions for Quartz.NET — job-execution trace correlation, the clustered Postgres store with its execution history and hosted-scheduler defaults, and Serilog job and trigger listeners — each scoped to a single concern and shipped independently on NuGet under `JorgeCostaMacia.Quartz.*`. Moved out of **shared-net** with the 10.2 versions, keeping the package ids and the version line. An independent project, not affiliated with Quartz.NET: the README says so, and the repo name must never read as the scheduler itself.

## Layout

- `src/<Package>/` — one package per folder. `test/<Package>.Tests/` — its tests (fakes, fixtures and harnesses in `Support/`, the token of the run a required last parameter of every async helper), plus `<Package>.IntegrationTests/` where a suite needs a real dependency: `Quartz` (Postgres for the store configuration — Quartz 4 validates the schema on start, so nothing about it can be asserted from a fake) and `Quartz.Serilog` (a live in-memory scheduler, no container). `assets/` — icons + social preview.
- **3-tier `Directory.Build.props`**: **root** (repo identity — Authors / Company / Copyright / Repository — + the single lockstep `VersionPrefix`; TFM `net10.0`; ImplicitUsings, Nullable, AnalysisLevel, EnforceCodeStyleInBuild, TreatWarningsAsErrors, the NuGet audit; `IsPackable` false by default) → **`src/`** (`IsPackable` true, package-output: icon / readme / license, SourceLink, symbols, `GenerateDocumentationFile`, pack of LICENSE/COPYRIGHT/icon/README) → **`test/`** (test settings). Each `src` csproj declares **only** `Description` / `PackageTags`; everything else is inherited from the props (don't restate it).

## Targets & stack

- TFM: **`net10.0`**, single-target, which is what Quartz 4 needs (it ships no net8/net9 asset). **Do not reintroduce a per-TFM conditional** — if a dependency needs one, the answer is to move the whole repo, not to split it.
- Quartz: **4.7.2** (4.7.1 and 4.7.2 change no API and no schema). 4.7 adds one **required** nullable column, `triggers.fixed_delay` (Quartz's `database/migrations/4.7/add_trigger_fixed_delay`): a 4.7 node refuses to start without it, and the migration runs safely while 4.6 nodes are up, so every store is migrated first and the nodes roll after — a worker that only parks retries included. A 4.6 node runs a fixed-delay trigger at a fixed rate, so `MeasureIntervalFromCompletion()` waits until every node of a store is 4.7. 4.7 also writes 0 for an unset seconds field in `CronExpressionBuilder` (nothing here uses it). Before it, 4.6 (4.6.1 and 4.6.2 change no API and no schema; since 4.6.1 a scheduler start no longer resets a declared trigger's fire count, a retry pending at shutdown or a pause an operator set). 4.6 adds three **required** nullable columns (`scheduler_state.quartz_version`, `fired_triggers.interrupt_requested_time`, `triggers.misfire_threshold`, from Quartz's `database/migrations/4.6`): a 4.6 node refuses to start without them and a 4.5 node never names them, so every store is migrated first and the nodes roll after, in any order. That includes the workers that only park retries: parking writes `triggers`. Once every live node checks in as 4.6, Postgres fires rounds without the `TRIGGER_ACCESS` lock (`quartz_version` coordinates it; `quartz.jobStore.lockFreeFiringRounds=false` turns it off), and the default misfire handling is `SmartPolicy`. Going net10-only is what unblocked 4.x (Quartz 4 ships no net8/net9 asset, so the multi-target could not reference it at all — `NU1202`). Five of 4.0's breaking changes bite here: listener members return **`ValueTask`**, `ITriggerListener.TriggerMisfired` takes an **`IScheduler`**, `IJob.Execute` takes a **`CancellationToken`** and returns `ValueTask`, `IJobExecutionContext` lost `Get`/`Put` — the per-firing bag is now `MergedJobDataMap` (a copy, persisted nowhere), which is what `JobTrace` uses — and `StdSchedulerFactory` is gone. **The listener ones fail silently**: every `IJobListener`/`ITriggerListener` member has a default implementation, so a `Task`-returning or short-signature member compiles, the default runs and the listener is never called. Quartz 4 guards it at `AddJobListener` and the integration tests are what catch it — never register a listener without one. The `StdSchedulerFactory` removal takes configuration by property strings and reflection with it: a test or console app now builds a scheduler with `QuartzSchedulerBuilder.Create(q => q.ConfigureScheduler(...).UseDefaultThreadPool(n).UseInMemoryStore()).BuildScheduler()`. `WithPostgresDefaults` also turns on **`UseExecutionHistory()`**: one execution and misfire history per cluster, in the store's own tables. How much it keeps is the host's: Quartz's bounds apply until it passes `options.WithDefaults()` (15 days, 50 000 entries per scheduler) to its own `AddQuartzExecutionHistory`. Quartz validates those tables, and the 4.3 and 4.4 columns, when the scheduler starts, so a store created before 4.4 needs Quartz's `database/migrations/4.2`, `4.3` and `4.4` scripts run first — the start fails naming the missing one. The 4.4 one (`add_execution_outcome_postgres.sql`) adds seven nullable columns to the execution history and a `JOB_STATUS` table; written for the `QRTZ_` prefix, so with a schema prefix such as `quartz` the tables are `quartz.execution_history` and `quartz.job_status`. A 4.3 node shares the store safely: it leaves the new columns null and never touches `JOB_STATUS`. Since 4.3 a persistent store also batches what is due (`MaxBatchSize` is automatic) and so takes `TRIGGER_ACCESS` to acquire as well as to fire.
- Tests: **xUnit v4 (the `xunit.v3` package, 4.x) on Microsoft.Testing.Platform v2 (MTP)** — test projects are `OutputType=Exe`, `dotnet test` runs MTP because the root **`global.json`** opts in, and `xunit.runner.visualstudio` is referenced for IDE discovery, as `testing-dotnet` sets up. Running them needs the **.NET 10 SDK**.
- Source is **UTF-8 without BOM** (`.editorconfig` `charset = utf-8`). camelCase locals, PascalCase types, I-prefixed interfaces. Copyright year stays **2023** (deliberate — don't bump).
- **Explicit types everywhere — spell the type out.** Never `var`, never target-typed `new()`, never collection expressions `[]`: write `new Foo(...)`, `new byte[] { ... }`, `new List<T> { ... }`, `Array.Empty<T>()`. The `.editorconfig` sets all three to explicit, but only `var` is analyzer-enforced — the analyzer never reports the implicit forms. Develop (`dotnet-build-nuget` from github-workflows) guards target-typed `new(` and implicit `new[]` with a **grep step** that fails the build, and collection expressions `[…]` with a second one (`code-style-dotnet`'s pattern: attributes, indexers, array types and list patterns never match it). Do not introduce any of them.

## Inter-package dependencies

`Quartz.Serilog` references `Quartz` via **`ProjectReference`**: `dotnet pack` turns it into a NuGet `<dependency>` at the same version, so both build against local source and **release together**. Nothing here depends on another `JorgeCostaMacia.*` repo — that is what lets this one release on its own — and `Quartz.Serilog` takes the third-party `Serilog`, not `JorgeCostaMacia.Serilog`. Keep it that way.

## Extend the framework's options, never replace its call

A package that carries a policy publishes it as an **extension on the type the framework already hands the host** — `IQuartzBuilder.WithPostgresDefaults()`, `ExecutionHistoryOptions.WithDefaults()`, `QuartzHostedServiceOptions.WithDefaults()` — and never as an `Add…` method that wraps the framework's own. The host keeps writing `AddQuartz(…)`, so what it composes stays readable in its `ProgramBuilder`; only the policy moves here. `http-dotnet` follows the same rule with its `ApiVersioningOptions` / `ProblemDetailsOptions` extensions, and its doc comments say so.

`Quartz.Serilog`'s `AddSerilogContext()` is not such a facade: it registers the package's own two listeners and wraps no framework call — the host still writes `AddQuartz(…)` and attaches them there with `AddJobListener` / `AddTriggerListener`. It is the package's composition root, the `Add<Module>Context` of `clean-architecture-dotnet`.

## Dependencies — Central Package Management

Third-party package versions are centralized in **`Directory.Packages.props`** (repo root, `ManagePackageVersionsCentrally=true`): add or bump them **there** as `<PackageVersion>`, and reference packages in csproj **without** a `Version`. (Inter-package deps are `ProjectReference`, not packages — see above — so CPM doesn't manage them.)

**The strict gates are on**: `TreatWarningsAsErrors` and the NuGet vulnerability audit on every restore (direct and transitive, every severity), so any warning — an advisory included — fails the build. The fix is a version bump or a transitive override in CPM, never `NoWarn`. The root `nuget.config` clears every inherited source and keeps nuget.org alone: these packages consume nothing else, and an inherited second source would raise `NU1507`. It is **deny-read** for agents — hand changes to it to the user.

## CI / publishing

The three workflows call the reusable ones of [github-workflows](https://github.com/JorgeCostaMacia/github-workflows) (`@v1`): `develop.yml` runs `dotnet-build-nuget` on develop and PRs (the explicit-types and collection-expression greps, build, the `dotnet format` check, test, pack — no publish); `main.yml` runs the same build on a push to `main` and then pushes the `nuget` artifact to nuget.org with Trusted Publishing (`NuGet/login`, OIDC) — that job stays here because the nuget.org policy names this repo's `main.yml`; `release.yml` creates the GitHub Release of each `v*` tag. **The (central) `VersionPrefix` is the publish gate**: `--skip-duplicate` leaves an existing version alone, so only a release that moved it publishes.

## Branching & releases — GitFlow

Use the **`gitflow` skill** for any branch/release work — never improvise.

- Feature/bugfix → `feature/`|`bugfix/<name>-<ts>` from develop → finish `--no-ff` into develop.
- Release → `release/<version>` from develop → bump the **single** `VersionPrefix` in the **root** `Directory.Build.props` (the one holding the solution identity — all packages version in lockstep) → Release Finish (merge develop+main, annotated tag `v<version>`, atomic push). One bump versions everything; the `ProjectReference` cross-deps follow automatically.
- Use git's **default merge message** (`--no-ff --no-edit`, never `-m`).
- Branch prefixes only: `feature` / `bugfix` / `release` / `hotfix`.

## Git etiquette

- Commit under **your own identity** — don't hardcode anyone's name/email.
- Keep history clean — **no** `Co-Authored-By` / AI-assistant trailers in commits or messages.
- Merges use git's **default** message (see *Branching & releases* above).

## Relevant skills

Skills that apply to this repo — let them trigger, or invoke explicitly. The first group is from `jorgecostamacia-agent-skills` (each `-dotnet` skill brings the generic one it realizes); the rest from `dotnet-agent-skills` (the `dotnet/skills` marketplace).

A skill listed here but missing on this machine is one command away: `claude plugin marketplace add JorgeCostaMacia/skills` (or `dotnet/skills`) once, then `claude plugin install <skill>@<marketplace>`.

- **`logging-dotnet`** (with **`logging`**) — the logging style `Quartz.Serilog` implements: fixed messages, every variable value through the log context, correlation in every scope — the `AggregateId`/`CorrelationId` pair `JobTrace` carries is that correlation.
- **`bus-dotnet`** — the main consumer of the store: the bus parks its retries in Quartz and a separate jobs worker runs them, on the store `WithPostgresDefaults` configures. A store change (a Quartz schema migration, a new column Quartz validates) reaches every worker that only parks retries too, since parking writes `triggers`.
- **`clean-architecture-dotnet`** (with **`clean-architecture`**) / **`solid`** — `AddSerilogContext()` as the package's `Add<Module>Context` composition root, internal by default with one public entry point, and design review of the public surface.
- **`code-style-dotnet`** — how the C# is written: explicit types, the 120-column width with declarations and constructions vertical, comments wrapped too, the gates.
- **`build-dotnet`** — the three-tier `Directory.Build.props`, the single `VersionPrefix`, CPM, the three-day package age, the NuGet audit as a build gate, the library's packing metadata, and the `nuget.config` base.
- **`testing-dotnet`** (with **`testing`**) — xUnit v3 on Microsoft.Testing.Platform, the test project layout, `TestContext.Current.CancellationToken`, the Testcontainers Postgres fixture and the in-memory scheduler harness.
- **`gitflow`** — all branch/release work.
- **`dotnet`** — C# language server + general .NET development.
- **`dotnet-msbuild`** — `Directory.Build.props`, project-file quality, CPM.
- **`dotnet-nuget`** — dependency management.
- **`dotnet-test`** / **`dotnet-test-migration`** — tests; the xUnit v4 / MTP v2 setup.
- **`dotnet-upgrade`** — target-framework migrations.

Not relevant to this repo (skip): `ddd` / `ddd-dotnet`, `validation-dotnet` (no value objects, aggregates or FluentValidation here — `JobTrace` carries plain identifiers), `efcore-dotnet` / `efcore-postgres-dotnet` (the store is Quartz's own ADO.NET job store, not EF Core), `messaging` / `bus-kafka-dotnet` (no messages here), `dotnet-aspnetcore` (that's http-dotnet), `dotnet-data`, `dotnet-ai`, `dotnet-maui`, `dotnet-blazor`, `dotnet-template-engine`, `dotnet11`, `dotnet-diag`, `dotnet-advanced`.

## Build & test

```
dotnet format quartz-dotnet.slnx                  # apply .editorconfig (using order, whitespace) — run before committing
dotnet build  quartz-dotnet.slnx -c Release
dotnet test   quartz-dotnet.slnx -c Release       # MTP v2 via global.json (needs the .NET 10 SDK + Docker); --logger is VSTest-only (MTP0001)
dotnet pack   quartz-dotnet.slnx -c Release        # packs all packable; tests are IsPackable=false
```

One suite needs **Docker running locally** — `Quartz.IntegrationTests` starts a Testcontainers Postgres, and without a daemon `dotnet test` fails there rather than skipping. `Quartz.Serilog.IntegrationTests` needs nothing external: it drives a real but in-memory Quartz scheduler.

One runtime requirement the packages cannot declare: `WithPostgresDefaults` makes Quartz resolve its ADO provider **by name at runtime**, so a host calling it must reference `Npgsql` itself. `JorgeCostaMacia.Quartz` deliberately does not depend on it — a consumer that only wants `JobTrace` should not pull in a database driver — and a host that forgets fails at startup with `ArgumentException: Error while reading metadata information for provider 'Npgsql'`.

Run **`dotnet format` before committing** — it applies the `.editorconfig` (using ordering, whitespace, line endings), the CLI equivalent of Visual Studio's *Code Cleanup*, and it is the normalizer for a `.cs` a shell tool wrote with LF. Develop checks it after the build with `dotnet format --verify-no-changes --no-restore`: the solution is clean, so the check only ever fails on what the push itself brought. `.gitattributes` checks `*.cs` out as CRLF (`eol=crlf`), so the Linux runner sees what the `.editorconfig` requires.
