# Loki-ARM32-Atomic-Operations-Patch---Raspberry-OS
Patch for : panic: unaligned 64-bit atomic operation 
# Loki ARM32 Atomic Operations Patch

## Overview

This repository contains an experimental patch intended for **Grafana Loki 3.6.16 running on a 32-bit ARM architecture**.

The patch aims to work around a potential issue involving atomic operations on 64-bit integers, which may cause the following panic:

```text
panic: unaligned 64-bit atomic operation
```

This issue was encountered while testing Loki on a Raspberry Pi 3B+ running 32-bit Raspberry Pi OS.

**Status: Experimental.** The patch has been verified to apply to the original source file. However, its effectiveness in preventing the panic has not yet been confirmed through runtime testing.

## When might this patch be useful?

This patch may be relevant when all the following conditions apply:

- Loki is compiled and run for the 32-bit `linux/arm` architecture.
- Loki crashes with the `unaligned 64-bit atomic operation` panic.
- The issue occurs while processing Loki query statistics, particularly in the `pkg/logqlmodel/stats` package.

This is not a general-purpose fix for all Loki issues. It is not intended for 64-bit ARM (`arm64`), 64-bit x86 (`amd64`), or unrelated errors.

## Why might this issue occur?

Loki uses atomic operations to update several statistics counters, including the number of processed lines, transferred bytes, and processing times.

Atomic operations on 64-bit integers can have memory-alignment requirements depending on the architecture and implementation.

On certain 32-bit ARM configurations, such an operation may fail if the required memory alignment is not satisfied.

The panic indicates that an atomic operation on a 64-bit integer was attempted with incorrect memory alignment. This patch attempts to avoid the affected atomic operations for the relevant counters when running on 32-bit ARM.

## What does the patch change?

The patch modifies the following source file:

`pkg/logqlmodel/stats/context.go`

It introduces three main changes.

### 1. Adds an `addInt64` function

The patch introduces a helper function for updating an `int64` counter.

- On 32-bit ARM, it uses a mutex (`c.mtx`) to protect the counter update.
- On other architectures, it continues to use `atomic.AddInt64`.

The goal is to replace the affected atomic operations with mutex-protected updates on 32-bit ARM.

### 2. Protects result collection

The `Result()` function also acquires the mutex on 32-bit ARM while reading and merging statistics.

This is intended to prevent concurrent access from conflicting with the new counter-update mechanism.

### 3. Replaces the affected atomic calls

Several calls to `atomic.AddInt64` in the statistics context are replaced with calls to `c.addInt64`.

The affected counters include statistics related to:

- Ingester data
- Chunks and indexes
- Compressed and decompressed bytes
- Processed lines
- Caches
- Queries and downloaded pages

## Expected impact

On 32-bit ARM, the patch replaces the affected atomic operations with mutex-protected updates. This may prevent the panic if those operations are indeed its cause.

However, this approach introduces synchronization and may reduce performance when multiple goroutines update statistics concurrently.

On other architectures, the code retains the `atomic.AddInt64` implementation.

**These effects still need to be measured. The patch is not guaranteed to eliminate the panic or improve performance.**

## Applying the patch

The patch is provided as:

`loki-arm32-atomic.patch`

Apply it to the source code matching the targeted Loki version. Back up any existing modifications and ensure that the target source file has not already been changed.

From the root of the Loki source repository, run:

```bash
git apply --check loki-arm32-atomic.patch
```

If the check succeeds, apply the patch with:

```bash
git apply loki-arm32-atomic.patch
```

A successful patch check does not replace compilation and runtime testing.

## Testing requirements

The following tests are still required to validate the patch:

1. Compile Loki for 32-bit ARM with the patch applied.
2. Start Loki and check its readiness.
3. Execute queries that exercise Loki's statistics handling.
4. Check whether the panic reappears in the logs.
5. Verify that counters and query results remain correct.
6. Evaluate any impact on performance and concurrent access.

## Limitations

- This is an experimental patch and is not presented as an official Grafana Loki fix.
- It modifies the statistics operations in `context.go`; it may not address every possible cause of the panic.
- Successful application does not guarantee correct runtime behavior.
- Compatibility with other Loki versions has not been established.

## Original test environment

- **Software:** Grafana Loki
- **Source version:** 3.6.16
- **Target architecture:** 32-bit ARM (`linux/arm`)
- **Test hardware:** Raspberry Pi 3B+
- **Operating system:** 32-bit Raspberry Pi OS

## License and attribution

This patch modifies code from the Grafana Loki project. Before redistributing it, check the original project's license and retain any applicable copyright notices.

Original project: https://github.com/grafana/loki
