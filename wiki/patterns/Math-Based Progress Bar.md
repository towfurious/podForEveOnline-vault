---
title: Math-Based Progress Bar
type: pattern
tags: [pattern, skill-queue, industry-job, layer-ui, layer-domain]
aliases: [Time-Based Progress Bar]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Math-Based Progress Bar

## When to use
You need a ticking progress bar or countdown for any time-bounded event whose `start` and `end` are already known. Applies directly to [[Skill Queue]] and [[Industry Job]]; would also apply to any future timed operation (market orders with expiries, contract windows, etc.).

**Do not use** when the endpoint offers no reliable end time, or when the progress is non-linear in time (e.g. extractor yields per cycle — decay curves — which are **not** linear and need a different model).

## Shape
- Fetch the event's `start: Instant` and `end: Instant` once.
- A domain `Clock` (`kotlinx.datetime.Clock.System` by default, injectable for tests) provides `now()`.
- A single coroutine on the screen emits ticks (`tickerFlow(1.seconds)` or `while { delay(1.seconds); emit(Unit) }`).
- Each tick, compute:
  - `progress = clamp((now - start) / (end - start), 0.0, 1.0)`
  - `remaining = end - now`
- Render the bar from `progress`; render the label from `remaining`.
- **Never** fetch network per tick. Re-fetch only on known shape-change events.

## Example

```kotlin
// shared / commonMain
class SkillProgressCalculator(private val clock: Clock = Clock.System) {
    fun snapshot(start: Instant, end: Instant): Snapshot {
        val now = clock.now()
        val total = (end - start).inWholeMilliseconds.coerceAtLeast(1)
        val elapsed = (now - start).inWholeMilliseconds.coerceIn(0, total)
        val progress = elapsed.toDouble() / total
        val remaining = (end - now).coerceAtLeast(0.seconds)
        return Snapshot(progress = progress, remaining = remaining)
    }
    data class Snapshot(val progress: Double, val remaining: Duration)
}
```

```kotlin
// composeApp — inside a screen Composable
val snapshot by produceState(initialValue = calc.snapshot(start, end), start, end) {
    while (true) {
        value = calc.snapshot(start, end)
        delay(1_000)
    }
}
LinearProgressIndicator(progress = { snapshot.progress.toFloat() })
Text(snapshot.remaining.formatHms())
```

## Anti-patterns
- **Polling ESI per tick** — wasteful, rate-limited, breaks offline.
- **Using device wall-clock directly without a `Clock` abstraction** — makes the math untestable; always inject `Clock`.
- **Re-fetching on every recomposition** — blows up network usage and thrashes the cache.
- **Using this for non-linear progress** (e.g. PI extractor yield curves).

## Related
- [[ADR-005 - Math-Based Skill Progress]] — the decision behind this pattern.
- [[Skill Queue]], [[Industry Job]] — entities using it.
- [[ADR-006 - Android Foreground Service]] — Android side reuses the same math to drive the shade countdown.
