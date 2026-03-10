---
name: fs2
description: Guides Scala FS2 functional stream processing — lazy, resource-safe, concurrent streaming built on Cats Effect. Use when building pipelines that read, transform, merge, or sink data with backpressure.
---

# FS2 — Functional Streams for Scala

## Quick start

- A `Stream[F, O]` is a **description** — nothing runs until you `compile` it.
- Use `evalMap` / `evalTap` for effectful steps, not `map` (which is pure).
- End every pipeline with a `compile` terminal: `.compile.drain`, `.compile.toList`, `.compile.fold(…)(…)`, etc.
- Wrap acquire/release in `Stream.bracket` or `Stream.resource` — the stream guarantees finalization even on error or cancellation.
- Prefer `parEvalMap` / `parEvalMapUnordered` over manual fiber management for bounded concurrent work.

## Workflow

1. **Source** — `Stream.emit`, `Stream.eval`, `Stream.iterate`, `Stream.resource`, `Files[IO].readAll`, `Stream.fromQueueUnterminated`
2. **Transform** — `map`, `filter`, `evalMap`, `evalFilter`, `through(pipe)`, `chunks`
3. **Compose** — `++`, `merge`, `concurrently`, `parJoin`, `zip`
4. **Compile** — `compile.drain`, `compile.toList`, `compile.fold`, `compile.count`, `compile.last`
5. **Run** — The resulting `F[…]` (usually `IO[…]`) is executed at your program's edge.

## Key rules

- **Effects belong in `eval*` operators** — `evalMap`, `evalTap`, `evalFilter`. Using `map` to run side effects silently discards them.
- **Resource safety** — prefer `Stream.bracket` / `Stream.resource` over manual acquire/release; finalization is guaranteed.
- **Concurrency** — `merge` interleaves two streams; `parJoin(n)` fans in a stream-of-streams with bounded concurrency; `parEvalMap(n)(f)` maps with bounded parallelism.
- **Error handling** — use `handleErrorWith` for recovery, `attempt` to surface errors as `Either`, `onFinalize` for cleanup.
- **Chunking** — FS2 processes data in `Chunk`s for performance; use `chunkN`, `unchunks`, `chunks` when batch size matters.
- **Backpressure** — built-in through bounded queues (`Queue.bounded`) and `parEvalMap`'s concurrency bound.
- **Common pitfall** — forgetting `.compile.drain` means nothing executes; a `Stream` is just a description.

## References

- See `references/fs2.md` for detailed API surface, concurrency primitives, I/O, error handling, and examples.
- Related skills: `cats-effect-io` (effect handling, fibers), `cats-effect-resource` (resource lifecycle).
  Install with: `npx skills add https://github.com/rlecomte/skills --skill fs2`
