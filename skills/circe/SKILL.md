---
name: circe
description: Scala JSON processing with Circe — parsing, encoding/decoding, cursor traversal, codec derivation (auto, semi-auto, custom), ADT handling, optics, and configuration. Use when working with JSON in Scala projects using the Circe library.
---

# Circe — JSON for Scala

## Quick start

- Circe uses `Encoder[A]` and `Decoder[A]` type classes to convert between Scala types and JSON.
- Parse JSON strings with `io.circe.parser.parse` (returns `Either[ParsingFailure, Json]`).
- Use `.asJson` (from `io.circe.syntax._`) to encode, `.as[T]` to decode.
- Prefer **semi-auto derivation** (`deriveEncoder`/`deriveDecoder`) for production code — explicit, no surprises.
- Use **auto derivation** (`io.circe.generic.auto._`) for quick prototyping only — implicit scope pollution and compile-time cost.
- For custom JSON shapes, use `Encoder.instance` / `Decoder.instance` or `forProductN` helpers.

## Workflow

1. **Parse** — `parse(jsonString)` returns `Either[ParsingFailure, Json]`; use `decode[T](jsonString)` to parse + decode in one step.
2. **Traverse** — Use `HCursor` with `.downField("key")`, `.downArray`, `.as[T]` to extract nested values.
3. **Encode** — Call `.asJson` on any value with an implicit `Encoder` in scope.
4. **Decode** — Call `.as[T]` on `Json` values, or `decode[T](string)` for direct string decoding.
5. **Derive codecs** — Use `deriveEncoder[T]`/`deriveDecoder[T]` (semi-auto) or `import io.circe.generic.auto._` (auto).

## Key rules

- **Parsing is separate** — `circe-core` has no parser; add `circe-parser` (JVM uses Jawn, JS uses `JSON.parse`).
- **Semi-auto over auto** — `deriveEncoder`/`deriveDecoder` gives explicit control, better compile times, no implicit ambiguity.
- **Cursor navigation** — `HCursor` tracks history; `ACursor` can represent failure. Use `downField`, `downArray`, `get[T]("field")`.
- **ADT encoding** — Default generic derivation wraps variants in a type discriminator object. Use `Configuration.default.withDiscriminator("type")` from `circe-generic-extras` for a flat discriminator field.
- **Accumulating errors** — Use `Decoder.accumulatingInstance` with `.asAcc[T]` and `.mapN` (from Cats) to collect all errors instead of failing fast.
- **Custom key mappings** — Use `Configuration.default.withSnakeCaseMemberNames` or `forProductN` for field name transformations.
- **Recursive ADTs** — Use `Decoder.recursive` to avoid stack overflows and diverging implicits.
- **Scala.js caveat** — `Long` values may lose precision due to `JSON.parse` converting to JavaScript numbers. Represent large numbers as strings if precision matters.

## References

- See `references/circe.md` for detailed API surface, all derivation patterns, cursor operations, ADT encoding, optics, and code examples.
