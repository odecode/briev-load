# briev-load

A Briev-native load and endurance harness: an arrival scheduler, bounded
attempt accounting with latency histograms, a process/CPU/RSS collector, and a
protocol-neutral nonblocking HTTP/WebSocket client transport.

This repository is the **engine**. It contains no endpoints, datasets, response
schemas or SQL — those belong to the application under test. An application
consumes the engine and supplies its own journey (see *Scenario contract*).

## Requirements

- A Briev compiler with its matching stdlib. Export `BRIEVC` (the compiler
  executable) or set `BRIEV_ROOT` to the compiler checkout (default
  `../briev-lang`).
- Linux x86-64 (the collector reads `/proc`; the transport uses Linux sockets).
- For the transport's mixed integer-width and entropy paths, use a compiler
  that already carries the journey integration fixes; stock `main` at the time
  of writing does not. This is a compiler-side requirement, not an engine one.

```sh
export BRIEVC=/path/to/brievc
export BRIEV_ROOT=/path/to/compiler       # for stdlib/config, if BRIEVC is elsewhere
bash check.sh
```

## Modules

| File | Responsibility |
|---|---|
| `lib/schedule.bv` | Fixed-rate arrival deadlines, bounded dispatch batches, explicit missed arrivals |
| `lib/metrics.bv` | Attempt lifecycle, outcome counters, latency histograms (total/service/lag) |
| `lib/process_metrics.bv` | Bracket `/proc` reads with monotonic time; CPU/RSS/fault deltas by identity |
| `lib/collect.bv` | Streaming 19-column process CSV; missed samples and read errors explicit |
| `lib/http_attempt.bv` | Protocol-neutral bounded HTTP/1 exchange: send a request, frame the response, hand the body back |
| `lib/transport.bv` | Incremental nonblocking HTTP + WebSocket client: upgrade, buffered frames, ping/pong |

The scheduler and accounting primitives are validated by `tests/*_gate.bv` plus
the `*_limits_gate` negative cases; `check.sh` also runs a live collector schema
check. `check-memory.sh` runs the Valgrind memory gate.

## Scenario contract

The engine deliberately has an inversion point: it owns *time*, *capacity* and
*measurement*; the application owns *what one unit of work is*.

An application scenario supplies:

- **Request bytes** for `http_attempt_start`/`http_attempt_progress` (the engine
  never builds a URL).
- **A success predicate** over `http_attempt_body(slot)`; when the response is
  framed with status 200, the engine returns `1` and the scenario decides
  `OUTCOME_OK` vs `OUTCOME_VALIDATION` before calling `http_attempt_finish`.
- **A live protocol**: call `journey_transport_progress` once per ready slot and
  interpret the returned `WsMessage` with the application's own event oracle.
- **Any durable verification** (SQL, files, receipts) in its own scripts.

The transport validates a WebSocket upgrade by RFC 6455 meaning (101 status,
`Upgrade`/`Connection` tokens, `Sec-WebSocket-Accept` value), not by a specific
server's byte layout, so a conforming peer is accepted regardless of header
order or extra fields.

## Reporting

Histograms are bounded and mergeable; report counts, overflow and maxima, and
quantiles only from bin edges. A run should record compiler, binary, dataset and
configuration provenance alongside its counters. No credentials or session
tokens belong in output.

## Limits

No capacity, latency-objective or endurance claim is implied by this engine.
Those require a scenario, a target, and a documented run geometry. The engine
provides the measurement primitives, not a verdict.