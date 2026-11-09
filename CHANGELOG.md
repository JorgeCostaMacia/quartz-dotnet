# Changelog

What a consumer of these packages has to act on, newest first. Versions that only touched the docs, the build or the internals are not listed.

## 10.3.0

Built on Quartz 4.7, which validates one more column on start, `triggers.fixed_delay`. Run Quartz's `database/migrations/4.7/add_trigger_fixed_delay` on every store before the first 4.7 node starts its scheduler; a node still on 4.6 keeps running on a migrated store.

## 10.2.0

`JorgeCostaMacia.Quartz` and `JorgeCostaMacia.Quartz.Serilog` continue here from shared-net, with the same ids and namespaces and the code of 10.1.5: bump the version, nothing else changes.
