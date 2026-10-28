# Loki ARM32 atomic operations patch

An experimental patch for Grafana Loki 3.6.16 on 32-bit ARM. I wrote it after Loki crashed on my Raspberry Pi 3B+ (32-bit Raspberry Pi OS) with this panic:

```text
panic: unaligned 64-bit atomic operation
```

**Status: experimental.** The patch applies cleanly to the original source file, but I haven't tested it at runtime yet, so I can't say whether it actually stops the panic.

## Is this for you?

Probably only if all of these are true:

- you build and run Loki for 32-bit `linux/arm`
- Loki crashes with the panic above
- the crash happens while Loki handles query statistics (`pkg/logqlmodel/stats`)

It's not a general fix for Loki, and it's not meant for `arm64`, `amd64` or unrelated errors.

## What's going on

Loki keeps a number of statistics counters (processed lines, transferred bytes, processing times...) and updates them with atomic operations on 64-bit integers. On some 32-bit ARM setups, these operations require the integer to be properly aligned in memory. 
The panic indicates that a 64-bit atomic operation was attempted on a memory address that did not meet the required alignment. The patch attempts to avoid the affected atomic operations in the statistics code on 32-bit ARM, but the root cause has not yet been fully confirmed.
Instead of dealing with the alignment, the patch avoids the affected atomic operations on 32-bit ARM.

## What the patch changes

It only touches `pkg/logqlmodel/stats/context.go`:

1. **New `addInt64` helper.** On 32-bit ARM it updates the counter under a mutex (`c.mtx`). On every other architecture it still calls `atomic.AddInt64`.
2. **`Result()` takes the mutex too** on 32-bit ARM while it reads and merges the stats, so it doesn't conflict with the new updates.
3. **Calls to `atomic.AddInt64` are replaced** with `c.addInt64`. The affected counters cover ingester data, chunks and indexes, compressed and decompressed bytes, processed lines, caches, and queries and downloaded pages.

## Expected impact

On 32-bit ARM, mutexes replace atomics. If those atomics are really the cause, the panic should go away. The downside is extra synchronization, which could slow things down when many goroutines update stats at the same time. I haven't measured any of this, so treat both the fix and the performance cost as unconfirmed.

Other architectures keep the original `atomic.AddInt64` code.

## Applying the patch

Run these from the root of the Loki repository, on source matching the targeted version. Back up any local changes first and make sure `context.go` hasn't already been modified.

```bash
git apply --check loki-arm32-atomic.patch
git apply loki-arm32-atomic.patch
```

A successful check only means the patch applies. It doesn't tell you the result compiles or runs correctly.

## Still to test

1. Build Loki for 32-bit ARM with the patch applied
2. Start Loki and check that it becomes ready
3. Run queries that exercise the statistics code
4. Check whether the panic still shows up in the logs
5. Make sure counters and query results are still correct
6. Look at the impact on performance and concurrent access

## Limitations

- This is not an official Grafana fix.
- It only changes the stats operations in `context.go`, so it may not cover every cause of this panic.
- Other Loki versions haven't been checked.

## Where I tried it

- Grafana Loki 3.6.16 (source)
- 32-bit ARM (`linux/arm`)
- Raspberry Pi 3B+ running 32-bit Raspberry Pi OS

## License

This patch modifies code from [Grafana Loki](https://github.com/grafana/loki).
