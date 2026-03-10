# FS2 Reference

## Table of Contents
- [Sources](#sources)
- [Core Types](#core-types)
- [Stream Creation](#stream-creation)
- [Transformation — Pure](#transformation--pure)
- [Transformation — Effectful](#transformation--effectful)
- [Pipes](#pipes)
- [Compilation Terminals](#compilation-terminals)
- [Resource Safety](#resource-safety)
- [Concurrency](#concurrency)
- [Concurrent Data Structures](#concurrent-data-structures)
- [I/O with fs2-io](#io-with-fs2-io)
- [Error Handling](#error-handling)
- [Chunking](#chunking)
- [Common Pitfalls & FAQ](#common-pitfalls--faq)
- [Test Prompts](#test-prompts)

---

## Sources

- Official site & guide: https://fs2.io
- GitHub: https://github.com/typelevel/fs2
- Scaladoc: https://www.javadoc.io/doc/co.fs2/fs2-core_3/latest/index.html
- fs2-io Scaladoc: https://www.javadoc.io/doc/co.fs2/fs2-io_3/latest/index.html

---

## Core Types

| Type | Purpose |
|---|---|
| `Stream[F, O]` | A lazy, pull-based sequence of `O` values producing effects in `F`. Nothing runs until compiled. |
| `Pipe[F, I, O]` | Alias for `Stream[F, I] => Stream[F, O]` — a reusable stream transformer. |
| `Pull[F, O, R]` | Low-level builder for stateful stream transformations. Prefer higher-level operators when possible. |
| `Chunk[O]` | An immutable, indexed sequence — the unit of batching inside a stream. |

---

## Stream Creation

```scala
// Pure
Stream.emit(1)                          // Stream[Pure, Int] — single element
Stream.emits(List(1, 2, 3))            // Stream[Pure, Int] — from collection
Stream.iterate(0)(_ + 1)               // Stream[Pure, Int] — infinite
Stream.range(0, 10)                    // Stream[Pure, Int]
Stream.empty                           // Stream[Pure, Nothing]
Stream.constant("a")                   // Stream[Pure, String] — infinite repeats

// Effectful
Stream.eval(IO(42))                    // Stream[IO, Int] — single effect
Stream.evals(IO(List(1, 2, 3)))        // Stream[IO, Int] — effect producing collection
Stream.repeatEval(IO(random()))        // Stream[IO, Double] — infinite effectful
Stream.exec(IO.println("hi"))          // Stream[IO, Nothing] — effect, no output

// Time-based (requires Temporal[F])
Stream.awakeEvery[IO](1.second)        // Stream[IO, FiniteDuration] — tick stream
Stream.fixedDelay[IO](500.millis)      // Stream[IO, Unit]
Stream.sleep_[IO](2.seconds)           // Stream[IO, Nothing] — delay then empty

// From resources
Stream.bracket(acquire)(release)       // guarantees release on completion/error/cancel
Stream.resource(myResource)            // from a cats.effect.Resource
Stream.fromAutoCloseable(IO(new FileInputStream("f")))
```

---

## Transformation — Pure

```scala
stream.map(f)                // transform elements
stream.filter(p)             // keep matching
stream.collect { case ... }  // partial function filter+map
stream.take(n)               // first n elements
stream.drop(n)               // skip first n
stream.takeWhile(p)          // take while predicate holds
stream.dropWhile(p)          // drop while predicate holds
stream.flatMap(f)            // f returns Stream; flattens
stream.fold(z)(f)            // fold to single value (emits once at end)
stream.scan(z)(f)            // running fold (emits each intermediate)
stream.zipWithIndex          // pair elements with 0-based index
stream.groupAdjacentBy(f)    // group consecutive elements by key
stream.intersperse(sep)      // insert separator between elements
stream.changes               // deduplicate consecutive equal elements
stream.head                  // first element only
stream.last                  // last element (wrapped in Option)
stream.chunkN(n)             // group into Chunk of size n
```

---

## Transformation — Effectful

```scala
stream.evalMap(f)            // f: O => F[O2] — effectful map (one-at-a-time)
stream.evalTap(f)            // f: O => F[Unit] — side-effect per element, keep original
stream.evalFilter(p)         // p: O => F[Boolean] — effectful filter
stream.evalMapChunk(f)       // like evalMap but preserves chunk structure
stream.evalMapAccumulate(s)(f) // stateful effectful map
```

> **Rule**: If `f` has effects (`IO`, `F[_]`), you **must** use an `eval*` operator. Using `map` with an effectful function compiles but silently discards the effect.

---

## Pipes

A `Pipe[F, I, O]` is just `Stream[F, I] => Stream[F, O]`. Use `through` to apply:

```scala
val toUpper: Pipe[IO, String, String] = _.evalMap(s => IO(s.toUpperCase))

stream.through(toUpper)
```

Built-in pipes from `fs2.text`:
```scala
stream.through(text.utf8.decode)   // Byte stream → String (handles split codepoints)
stream.through(text.utf8.encode)   // String → Byte stream
stream.through(text.lines)         // String stream → line-by-line
```

---

## Compilation Terminals

Compilation converts `Stream[F, O]` → `F[…]`. Nothing runs until you compile:

```scala
stream.compile.drain       // F[Unit] — run for effects, discard output
stream.compile.toList      // F[List[O]] — collect all output
stream.compile.toVector    // F[Vector[O]]
stream.compile.fold(z)(f)  // F[O2] — fold all output
stream.compile.count       // F[Long] — count elements
stream.compile.last        // F[Option[O]] — last element
stream.compile.lastOrError // F[O] — last element or raise error
stream.compile.onlyOrError // F[O] — exactly one element or raise
```

---

## Resource Safety

### Stream.bracket
```scala
Stream.bracket(IO(openFile(path)))(f => IO(f.close()))
  .flatMap(f => Stream.emits(readLines(f)))
```
Release runs on normal completion, error, or cancellation.

### Stream.resource
```scala
// Lift a cats.effect.Resource into a Stream
Stream.resource(Resource.make(IO(acquire))(r => IO(release(r))))
  .flatMap(r => useAsStream(r))
```

### Stream.fromAutoCloseable
```scala
Stream.fromAutoCloseable(IO(new BufferedReader(new FileReader("data.csv"))))
```

### Nested resources compose
```scala
for {
  db   <- Stream.resource(databasePool)
  file <- Stream.bracket(IO(openFile))(f => IO(f.close()))
  line <- Stream.emits(readLines(file))
  _    <- Stream.eval(db.insert(line))
} yield ()
```
All resources are released in reverse order of acquisition.

---

## Concurrency

### merge — interleave two streams
```scala
stream1.merge(stream2)  // elements from both, nondeterministic interleaving
// Both run concurrently; output stream completes when both inputs complete.
```

### parJoin — fan-in a stream of streams
```scala
val streams: Stream[IO, Stream[IO, Int]] = ???
streams.parJoin(maxOpen = 10)  // run up to 10 inner streams concurrently
streams.parJoinUnbounded        // no concurrency limit (use with care)
```

### parEvalMap — bounded parallel map
```scala
stream.parEvalMap(maxConcurrent = 8)(url => fetchUrl(url))
// Preserves output order. Use parEvalMapUnordered for unordered (faster).
```

### concurrently — run a background stream for its effects
```scala
mainStream.concurrently(backgroundStream)
// backgroundStream runs for effects only; its output is discarded.
// If backgroundStream fails, mainStream is interrupted.
```

### zip / zipWith — synchronize two streams
```scala
stream1.zip(stream2)              // pairs elements 1:1
stream1.zipWith(stream2)(_ + _)   // combine paired elements
```

---

## Concurrent Data Structures

These live in `fs2.concurrent` and bridge producers/consumers:

### Queue
```scala
for {
  q      <- Stream.eval(Queue.bounded[IO, Int](100))
  producer = Stream.iterate(1)(_ + 1).evalMap(q.offer)
  consumer = Stream.fromQueueUnterminated(q)
  result <- consumer.concurrently(producer).take(50).compile.toList
} yield result
```

### Topic — broadcast to multiple subscribers
```scala
for {
  topic <- Stream.eval(Topic[IO, String])
  pub    = Stream("a", "b", "c").covary[IO].through(topic.publish)
  sub    = topic.subscribe(maxQueued = 10)
  // Each subscriber receives all published elements
} yield ()
```

### SignallingRef — reactive state
```scala
for {
  sig    <- Stream.eval(SignallingRef[IO, Boolean](false))
  worker  = stream.interruptWhen(sig)  // stops when signal becomes true
  _      <- worker.concurrently(Stream.sleep_[IO](10.seconds) ++ Stream.eval(sig.set(true)))
} yield ()
```

### Channel — async element passing (closeable)
```scala
for {
  ch <- Stream.eval(Channel.bounded[IO, Int](10))
  _  <- Stream.eval(ch.send(1) >> ch.send(2) >> ch.close)
  out <- ch.stream.compile.toList  // List(1, 2)
} yield out
```

---

## I/O with fs2-io

Requires `fs2-io` dependency. Core entry point: `fs2.io.file.Files[IO]`.

### Reading files
```scala
import fs2.io.file.{Files, Path}

Files[IO].readAll(Path("data.txt"))        // Stream[IO, Byte]
  .through(text.utf8.decode)
  .through(text.lines)
```

### Writing files
```scala
stream
  .through(text.utf8.encode)
  .through(Files[IO].writeAll(Path("out.txt")))
  .compile.drain
```

### Watching for file changes
```scala
Files[IO].watch(Path("dir/"), modifiers = Nil)  // Stream[IO, Watcher.Event]
```

### Network (TCP)
```scala
import fs2.io.net.Network

Network[IO].client(SocketAddress(host"localhost", port"8080"))
  .use { socket =>
    socket.reads.through(text.utf8.decode).compile.string
  }
```

### Standard I/O
```scala
fs2.io.stdinUtf8[IO](4096)              // Stream[IO, String] from stdin
stream.through(fs2.io.stdoutLines())     // print to stdout
```

---

## Error Handling

```scala
// Recover from errors
stream.handleErrorWith(e => Stream.emit(fallback))

// Surface errors as Either
stream.attempt  // Stream[F, Either[Throwable, O]]

// Cleanup on finalization (normal, error, or cancel)
stream.onFinalize(IO.println("done"))

// Retry with backoff (common pattern)
Stream.retry(
  fo = fetchData,
  delay = 1.second,
  nextDelay = _ * 2,
  maxAttempts = 5
)
```

> **Rule**: Errors in a stream propagate downstream and terminate the stream unless caught by `handleErrorWith` or `attempt`.

---

## Chunking

FS2 internally batches elements in `Chunk`s for performance.

```scala
stream.chunks                 // Stream[F, Chunk[O]] — expose chunks
stream.unchunks               // Stream[F, O] — flatten chunks back
stream.chunkN(100)            // group into chunks of 100
stream.chunkN(100, allowFewer = true) // last chunk may be smaller

// Process by chunk for batch operations (e.g., bulk DB insert)
stream.chunks.evalMap(chunk => db.insertBatch(chunk.toList))
```

---

## Common Pitfalls & FAQ

| Pitfall | Fix |
|---|---|
| Using `map` with an effectful function — effect is silently discarded | Use `evalMap` / `evalTap` for any `F[_]` returning function |
| Forgetting `compile.drain` — stream never executes | Every pipeline must end with a `compile` terminal |
| Unbounded `parJoinUnbounded` / `Queue.unbounded` in production | Use bounded variants (`parJoin(n)`, `Queue.bounded(n)`) for backpressure |
| Using `unsafeRunSync()` inside `evalMap` | Compose with `flatMap`/`evalMap` instead; never run effects inside a stream unsafely |
| Resource leak from manual acquire/release | Use `Stream.bracket` or `Stream.resource` — finalization is guaranteed |
| Large `compile.toList` on unbounded stream | Use `take(n)` or `fold`/`drain` — don't collect infinite streams into memory |
| Ignoring `text.utf8.decode` when reading bytes | Raw bytes may split multi-byte codepoints; always decode through `text.utf8.decode` |
| Mixing `IO.blocking` work in `evalMap` without `evalMapChunk` | For blocking I/O, the Cats Effect runtime handles thread shifting; ensure you wrap blocking calls in `IO.blocking` |

**FAQ**

**Q: When should I use `Pull` directly?**
A: Only when you need element-by-element stateful logic that can't be expressed with `scan`, `mapAccumulate`, `evalMapAccumulate`, or `groupAdjacentBy`. Most stream programs never need `Pull`.

**Q: How do I convert between fs2 Stream and other streaming libraries?**
A: Use interop libraries: `fs2-reactive-streams` provides `toUnicastPublisher` / `fromPublisher` for Reactive Streams interop. For Akka/Pekko Streams, go through Reactive Streams as the bridge.

**Q: How do I handle infinite streams safely?**
A: Use `take`, `takeWhile`, `interruptWhen`, or `interruptAfter` to bound them. Always use `compile.drain` (not `compile.toList`) for streams that shouldn't be collected.

---

## Test Prompts

1. "Read a CSV file with fs2, skip the header, parse each row, filter invalid rows, and write valid rows to an output file."
2. "Create a producer-consumer pipeline using fs2 Queue where 3 producers generate events and a single consumer processes them."
3. "Build an fs2 stream that polls an HTTP endpoint every 5 seconds and stops after 10 consecutive errors."
4. "Merge two fs2 streams of sensor readings and compute a running average, emitting every 100 readings."
5. "Use fs2 to watch a directory for new files and process each new file concurrently (max 4 at a time)."
