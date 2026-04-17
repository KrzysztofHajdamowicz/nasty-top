# nasty-top Tuning Rules

The advisor evaluates these rules on every tick (2s), in priority order.
The first matching rule produces a proposal shown in the footer bar.
User presses `Y` to apply the suggested sysfs write.

Rules are implemented in `src/advisor.rs`.

## Active Rules

| # | Condition | Proposal | Rationale |
|---|-----------|----------|-----------|
| 1 | Journal fill > 80% | Halve `journal_reclaim_delay` (min 10) | Journal is nearly full — reclaim space faster to prevent write stalls from journal exhaustion |
| 2 | Journal fill > 50% + watermark != "stripe" | Halve `journal_flush_delay` (min 100) | Journal filling with abnormal watermark — flush dirty entries more often to keep headroom |
| 3 | Write stalls (last 30s) + `copygc_enabled=1` + copygc active | Set `copygc_enabled=0` | Background copy-GC is competing with foreground writes causing latency spikes |
| 4 | Write stalls (last 30s) + `rebalance_enabled=1` + rebalance running | Set `rebalance_enabled=0` | Background rebalance IO is competing with foreground writes |
| 5 | Read stalls (last 30s) + `btree_cache_size_max=0` (auto) | Set `btree_cache_size_max=512M` | Read stalls with auto-sized btree cache — explicit larger cache may reduce btree read misses |
| 6 | 3+ stalls in 60s + `gc_reserve_percent` < 15 | Increase `gc_reserve_percent` by 4 (max 20) | Frequent stalls suggest allocator pressure — more GC reserve gives the allocator breathing room |

## Stall Detection

A "stall" is any device reporting read or write latency > 100ms (configurable via `STALL_THRESHOLD_NS` in `src/app.rs`).

Up to 10 recent stall events are tracked. The Background section turns red and lists the last 5 with device, direction, latency, and seconds ago.

## Future Rule Ideas

- High `blocked_allocate` count rate → increase `gc_reserve_percent` or `gc_reserve_bytes`
- High `blocked_journal_low_on_space` rate → reduce `journal_flush_delay`
- High `blocked_journal_max_in_flight` rate → reduce concurrent writers or increase journal size
- High `blocked_write_buffer_full` rate → tune write buffer settings
- Compression ratio dropping → suggest trying different compression algorithm
- Read amplification (btree reads >> user reads) → suggest larger btree node size (mount-time only)
- Device with significantly higher latency than others → flag potential hardware issue
