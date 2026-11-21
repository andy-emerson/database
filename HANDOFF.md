# HANDOFF: New Direction for TallyDB

This document captures the redesigned direction for TallyDB agreed in design discussions. The goal is an append-optimized, compute-optimized database for quant research and similar workloads, built by leaning on the existing ecosystem rather than hand-rolling every layer.

## Core Decisions

- **Storage**: Reuse the existing TallyDB Arrow-shaped persistent segment format (minimal serialization from storage to memory). No redesign of the on-disk layout.
- **Concurrency**: Single-writer per table (not database-wide). Concurrent reads are already supported and are sufficient. Multi-source tables are ordinary relational composition (joins/views).
- **Query / compute engine**: DataFusion. A custom `TableProvider` surfaces the existing segments. Column-level width uniformity is enforced for numeric columns at write time. Columns are classified as numeric vs other for storage and kernel selection. Specialized aggregate/window kernels are optional (no longer required for zero-copy reasons).
- **Server surface**: Arrow Flight (Flight SQL if the standard SQL-over-Flight path is desired). Existing crates (`arrow-flight`, related DataFusion Flight helpers).
- **Client compute**: datafusion-python for zero-copy Arrow → Python / NumPy / Pandas. Python (including NumPy/Pandas) only for now; Julia/R later if needed.
- **Embeddability**: Deprioritized. The primary model is a server database.
- **Admin / notebook features**: Markdown files containing `sql` and `python` code blocks are executed by the admin tool (not the core server). This keeps the core small and gives the admin tool a clear commercial feature.

## What the Foundation Consists Of

1. DataFusion + a `TableProvider` that maps existing segments.
2. Write-path coordination that turns appends into new immutable segments and updates the catalog so concurrent readers see them.
3. A small catalog (schemas, segment lists, statistics) kept consistent with the data files.
4. Arrow Flight (or Flight SQL) server in front of the SessionContext.
5. datafusion-python on the client side for zero-copy compute.

Most of the work is glue and integration. The existing segment format and concurrent-read model are treated as given.

## What Is Explicitly Deferred

- Full admin tool and Markdown notebook runner (lives outside the core).
- Specialized numeric kernels (add later only if profiling shows a win).
- Multi-writer concurrency.
- Non-Python language bindings.
- Hosted cloud service.

## Performance Expectations

The structural advantages that made the previous design fast (Arrow-shaped storage, in-engine / zero-copy compute, dense numeric columns) remain. The new design should stay substantially faster than a DuckDB + NumPy workflow on append-heavy, compute-in-place loads. Any regressions versus the old hand-tuned kernels can be addressed later with custom ExecutionPlans if needed. The clearer wins are maintainability and the fact that Python/NumPy/Pandas are now zero-copy citizens.

## Next Step

Stand up a minimal Flight server with a TableProvider over the existing segments. Everything else can be added without changing this foundation.
