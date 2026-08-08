# Meta
[meta]: #meta
- **Name:** Jackson 3 support for the Java SDK and Spring Boot starter
- **Start Date:** 2026-08-08
- **Author(s):** [@curfew-marathon](https://github.com/curfew-marathon) <!-- confirm handle before opening the PR -->
- **Status:** Draft
- **RFC Pull Request:** (leave blank)
- **Relevant Issues:**
  - https://github.com/openfga/java-sdk/issues/300 (see also #349)
  - https://github.com/openfga/spring-boot-starter/pull/183
- **Supersedes:** N/A

## Table of Contents
- [Summary](#summary)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [What it is](#what-it-is)
- [How it Works](#how-it-works)
- [Migration](#migration)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)

## Summary
[summary]: #summary

The OpenFGA Java SDK and Spring Boot starter are pinned to Jackson 2.x. Spring Boot 4.0 moves its auto-configuration baseline to Jackson 3.0 (the `tools.jackson.*` namespace) and ships Jackson 2 support only in a deprecated form. This RFC proposes migrating the SDK to Jackson 3 through a deprecation bridge: an SDK-owned `JsonSerializer` interface is introduced on the current Jackson 2 line as a non-breaking minor, the leaking `ObjectMapper` accessors are deprecated but kept working as delegating wrappers, and the switch to Jackson 3 happens in a later major. The result is that the roughly 99% of users who never touch a mapper see no change, and users who do get a full release of warning plus a one-page guide instead of a runtime surprise. The 0.x line is retained as a Jackson 2 / Spring Boot 3 LTS.

## Definitions
[definitions]: #definitions

- **Jackson 2 / Jackson 3:** major versions of the Jackson JSON library. Jackson 3 relocates its databind and core types from the `com.fasterxml.jackson.*` package namespace to `tools.jackson.*`. The two versions can coexist on one classpath because the namespaces are disjoint.
- **`ObjectMapper`:** Jackson's central serialization entry point. `com.fasterxml.jackson.databind.ObjectMapper` in Jackson 2; `tools.jackson.databind.ObjectMapper` (and the preferred `tools.jackson.databind.json.JsonMapper`) in Jackson 3. The two share no common supertype.
- **`JsonSerializer` (proposed):** an SDK-owned interface that hides the concrete Jackson mapper behind SDK-controlled methods (for example `byte[] writeValueAsBytes(Object)` and `<T> T readValue(byte[], Class<T>)`), so the SDK's public API no longer names a Jackson type.
- **Deprecation bridge:** a release that adds the new API *and* keeps the old one working as a delegating wrapper, rather than deleting the old one. The removal happens only at the subsequent major.
- **Static-safe surface:** API elements that do not change between Jackson 2 and 3. Jackson annotations are static-safe because Jackson 3 depends on the Jackson 2 annotations artifact.
- **`api` vs `implementation` scope (Gradle):** an `api` dependency is exposed on the consumer's compile classpath transitively; an `implementation` dependency is not. Jackson is currently `api`-scoped in the SDK, which forces Jackson 2 onto every consumer.
- **Target personas:** application developer (SDK consumer), project contributor (SDK/starter maintainer).

## Motivation
[motivation]: #motivation

### Why should we do this?

Issue [#300](https://github.com/openfga/java-sdk/issues/300) reports that the SDK's public API binds consumers to the Jackson 2 compile-time classpath, which conflicts with Spring Boot 4 / Jackson 3. Two surfaces cause it:

1. **The `ObjectMapper` accessors.** `ApiClient.getObjectMapper()`, `setObjectMapper(ObjectMapper)`, and the `ApiClient(HttpClient.Builder, ObjectMapper)` constructor expose `com.fasterxml.jackson.databind.ObjectMapper` on public signatures. In Jackson 3 that type changes namespace with no shared supertype, so a single signature cannot serve both.
2. **The transitive classpath leak.** `build.gradle` declares `jackson-core`, `jackson-annotations`, and `jackson-databind` as `api` scope, forcing Jackson 2 onto every consumer's compile classpath. Under a Spring Boot 4 BOM this collides with the Jackson 3 stack.

The Spring Boot starter is the concrete consumer feeling this: a Boot 4 application resolves Jackson to `tools.jackson:3.x`, while the SDK still requires Jackson 2, producing version conflicts.

### What use cases does it support?

- Spring Boot 4 applications using the OpenFGA starter and SDK.
- Any downstream Java application on the Jackson 3 line.

### What is the expected outcome?

A Jackson-3-capable SDK and starter where the common call path (`check`, `write`, `read`, `listObjects`) and the generated model classes are unchanged, existing users can upgrade without code changes if they heed one release of deprecation warnings, and Spring Boot 4 works out of the box.

## What it is
[what-it-is]: #what-it-is

The proposal ships a Jackson-3-forward single artifact, delivered through a deprecation bridge, with the SDK's public serialization surface abstracted behind an SDK-owned `JsonSerializer` interface so the common path and the models never break.

The key insight that makes this narrow: Jackson annotations do not move. Jackson 3 depends on the Jackson 2 annotations artifact (the 3.0 `pom.xml` states "Annotations remain at Jackson 2.x group id"). The roughly 84 `@JsonProperty` / `@JsonInclude` / `@JsonPropertyOrder` / `@JsonValue` / `@JsonCreator` usages across the generated models are therefore static-safe and stay user-facing unchanged. Only databind and core types (`ObjectMapper`, `TypeReference`, `JavaType`, `JsonProcessingException`, feature enums) change namespace.

From the perspective of the three personas:

- **The application developer who never touches a mapper** sees nothing change. The common call path exposes no Jackson databind type; the annotated models are unchanged.
- **The application developer who customizes the mapper** gets, in the bridge release, a deprecation warning pointing at the new `getJsonSerializer()` / `setJsonSerializer(...)` API. Their existing code still compiles and runs. Only at the Jackson 3 major is the old accessor removed, at which point they get a compile error (never a runtime failure) and a one-page migration guide. A 0.x LTS line remains as an escape hatch.
- **The Spring Boot starter maintainer** gains a dual `@ConditionalOnClass` configuration that hands the SDK a `JsonSerializer` built from whichever mapper Spring Boot auto-configured (`com.fasterxml...ObjectMapper` on SB3, `tools.jackson...JsonMapper` on SB4).

Example deprecation warning in the bridge release:

```
warning: [deprecation] getObjectMapper() in ApiClient has been deprecated
  use getJsonSerializer() instead; the ObjectMapper accessor will be removed
  when the SDK moves to Jackson 3 (see MIGRATION.md)
```

## How it Works
[how-it-works]: #how-it-works

Work is sequenced across three repositories, and the dependency chain fixes the order: `sdk-generator` templates, then `java-sdk`, then `spring-boot-starter`. Generated-file changes must originate in `sdk-generator` templates, or the next `sync/sdk-generator` PR reverts hand edits.

### Wave 0: the bridge (java-sdk, current Jackson 2 line, non-breaking minor)

- Introduce the SDK-owned `JsonSerializer` interface (`byte[] writeValueAsBytes(Object)`, `<T> T readValue(byte[]/String, Class<T>)`, plus a generic-read variant via an SDK-owned type token so no `TypeReference` / `JavaType` appears in signatures).
- Add `Jackson2JsonSerializer` implementing it, wrapping today's exact mapper config: `NON_NULL` inclusion, `FAIL_ON_UNKNOWN_PROPERTIES=false`, `FAIL_ON_INVALID_SUBTYPE=false`, dates-as-ISO, enums-as-`toString`, `JavaTimeModule`, `JsonNullableModule`.
- Route `ApiClient` and `FgaError` through the interface. Keep `getObjectMapper()` / `setObjectMapper(ObjectMapper)` / the `ObjectMapper` constructor as `@Deprecated` delegating wrappers: `setObjectMapper(m)` wraps `m` in a `Jackson2JsonSerializer`; `getObjectMapper()` unwraps back to a real Jackson 2 mapper. Add `getJsonSerializer()` / `setJsonSerializer(...)` and an `ApiClient(builder, JsonSerializer)` constructor.
- Add the SDK type-token streaming overload and deprecate the `TypeReference<StreamResult<T>>` overload plus the `BaseStreamingApi` `TypeReference` constructor.
- Wrap `throws JsonProcessingException` behind an SDK exception.
- Gate on a byte-for-byte wire-parity test: serialized output must be identical to the current mapper across representative models.
- Definition of done: existing tests pass unchanged, no public signature changes (only additions and deprecations), wire output identical. This release breaks nobody.

Idiom precedent already in the repo: `ApiClient.urlEncode` uses `@Deprecated(forRemoval=true, since=…)`.

### Wave A: generator templates (sdk-generator)

- Reproduce the interface routing for the two generated carriers (`api.mustache` → `OpenFgaApi`, `BaseStreamingApi.mustache`) so regeneration preserves it.
- Spike the generator's opt-in `useJackson3` flag against the `native` library and the custom templates. If it cleanly covers the deltas, drive the namespace switch from it; otherwise hand-patch the databind/core templates: `ObjectMapper` / `TypeReference` / `JsonProcessingException` → `tools.jackson.*`; `SerializationFeature.WRITE_DATES_AS_TIMESTAMPS` → `DateTimeFeature`; drop explicit `JavaTimeModule` registration; `JsonDeserializer` / `JsonSerializer` / `Module` → `ValueDeserializer` / `ValueSerializer` / `JacksonModule` in the oneOf/anyOf serializers and `CustomInstantDeserializer`.
- Leave the annotation partials untouched.
- Set the Jackson 3 BOM and `implementation` scope for databind/core in `build.gradle.mustache`, keeping `jackson-annotations` on `api`.
- Regenerate `java-sdk` in the same change.
- Definition of done: a clean regen reproduces the Jackson 3 client with no manual diff.

### Wave B: java-sdk Jackson 3 flip (new major)

- Add `Jackson3JsonSerializer` on `tools.jackson.databind.json.JsonMapper` (using `builderWithJackson2Defaults()` as an aid), mapping each Jackson 2 feature to its Jackson 3 equivalent (`DateTimeFeature`, built-in JavaTime, `EnumFeature`).
- Remove the deprecated `ObjectMapper` and `TypeReference` methods (or retype; see Unresolved Questions). Demote databind/core to `implementation`; keep `jackson-annotations` on `api`.
- Resolve the `JsonNullable` path (0 model usages today).
- Re-run the wire-parity gate under Jackson 3.
- Release the major with a one-page migration guide and CHANGELOG callout. Keep 0.x as a Jackson 2 / SB3 LTS for security patches.

### Wave C: spring-boot-starter (gated on the released SDK major)

- Split into `-autoconfigure` and `-starter` modules with optional Jackson deps.
- Add dual `@ConditionalOnClass` configuration: a Jackson 2 path injecting `ObjectProvider<com.fasterxml...ObjectMapper>` and a Jackson 3 path injecting `ObjectProvider<tools.jackson...JsonMapper>`, each handing the SDK a `JsonSerializer`. Replace `createDefaultObjectMapper()` with per-version factories.
- Keep explicit `com.fasterxml` Jackson coordinates rather than relying on `spring-boot-starter-json`, which resolves to Jackson 3 under a Boot 4 BOM and collides with the still-Jackson-2 SDK on prior waves.
- Bump `jackson-databind-nullable` to 0.2.10+ (dual J2/J3 via ServiceLoader SPI).
- Fix the two `PropertyMapper$Source.as(Function)` call sites (`toCredentials`, `toTelemetryConfiguration`), re-signatured in SB 4.0, by moving the conversion into the `.to(...)` lambda rather than removing `PropertyMapper`.
- Address the changed `testcontainers` dependency coordinates under the 4.1 baseline.
- Add a Boot 4 BOM row to the CI matrix (today it builds only the 3.4 baseline) plus an SB4 + Jackson 3 integration test, keeping the SB3 + Jackson 2 one.
- Definition of done: the same starter jar boots green on both SB3/Jackson 2 and SB4/Jackson 3.

## Migration
[migration]: #migration

**Public API breaks (application developers who customize the mapper):**

| Surface | Break | Mitigation |
| --- | --- | --- |
| `ApiClient.getObjectMapper()` | Removed at the Jackson 3 major | Deprecated one full minor earlier; migrate to `getJsonSerializer()` |
| `ApiClient.setObjectMapper(ObjectMapper)` | Removed at the major | Deprecated earlier; migrate to `setJsonSerializer(...)` or the `JsonSerializer` constructor |
| `ApiClient(HttpClient.Builder, ObjectMapper)` | Removed at the major | Deprecated earlier; use `ApiClient(builder, JsonSerializer)` |
| `OpenFgaClient.streamingApiExecutor(TypeReference<…>)` | Removed / retyped at the major | Use the SDK type-token overload (or `Class<T>`) added in the bridge |
| `throws JsonProcessingException` | Wrapped behind an SDK exception | Catch the SDK exception |

A user who acts on the bridge-release deprecation warnings traverses the entire migration with zero breaks. A user who skips the bridge gets a compile error (not a runtime failure) at the major, plus a one-page migration guide (import swaps for the mapper surface; a note that annotations are unchanged).

**No wire-format break for anyone:** the byte-for-byte parity gate (M3) runs in Waves 0 and B, so no user observes a changed payload (dates, null omission, field order, unknown-property tolerance).

**Transitive classpath (all consumers):** demoting databind/core to `implementation` removes Jackson 2 from consumers' compile classpath. Consumers who relied on that transitive dependency must declare Jackson directly; this is documented in the guide.

**SB3 / Jackson 2 users:** the 0.x line is retained as an LTS for security patches (see Unresolved Question 1 for the feature-parity decision).

**Spring Boot starter users:** no public bean exposes a Jackson type, so starter users are not directly broken; the change is internal wiring plus README/example updates.

## Drawbacks
[drawbacks]: #drawbacks

- Maintaining two serialization implementations (`Jackson2JsonSerializer`, `Jackson3JsonSerializer`) plus the interface adds surface area.
- A byte-for-byte parity test is a real maintenance and authoring cost, though it is what guarantees an invisible wire.
- The `JsonSerializer` abstraction adds one indirection for the small set of power users who legitimately want direct mapper access; they must go through SDK methods or drop to the 0.x LTS.
- If maintainers later need to keep shipping features (not just security fixes) to Jackson 2 users, this single-artifact approach forces a re-architecture toward adapter jars (see Alternatives).
- Cross-repo sequencing (generator → SDK → starter) means the starter's Spring Boot 4 support cannot land until the SDK major is released.

## Alternatives
[alternatives]: #alternatives

### Separate adapter jars over a shared SPI core (considered, held as escalation)

Split the SDK into `openfga-sdk-core`, `openfga-sdk-jackson2`, and `openfga-sdk-jackson3`. Cleanest coexistence story and the only option that keeps feature parity for Jackson 2 users indefinitely. Not chosen as the default because of cost: three published artifacts and dual code generation. Held as the escalation path if Unresolved Question 1 favors continued Jackson 2 feature work.

### Single artifact, runtime auto-detection of Jackson 2 or 3 (rejected)

One artifact declaring both Jacksons optional, selecting at runtime. Rejected because Jackson 2 and 3 share no common mapper interface, so the public `ObjectMapper` surface cannot be abstracted type-safely; the result is brittle and unsafe.

### Shade / relocate Jackson inside the SDK (rejected)

Relocate Jackson into an internal package so consumers never see it. Rejected because the annotations users rely on cannot be shaded (they are part of the public model surface), it adds binary bloat, and the SDK would then own patching every Jackson CVE.

### Jackson-3-only flip with no bridge (rejected)

Cut straight to Jackson 3 in a major with no deprecation window. Rejected because it turns a narrow, warned migration into an abrupt break for every mapper user at once, against the goal of a near-invisible upgrade.

### Impact of not doing this

The SDK and starter remain unusable on Spring Boot 4 as it becomes the default, stranding users on Spring Boot 3 and eventually on an unmaintained Jackson 2 line.

## Prior Art
[prior-art]: #prior-art

- **`jackson-databind-nullable` [PR #117](https://github.com/OpenAPITools/jackson-databind-nullable/pull/117)** delivers dual Jackson 2/3 support from a single artifact via a `ServiceLoader` SPI that auto-selects whichever Jackson is present: direct precedent for the coexistence model, and the mechanism the starter's nullable dependency (0.2.10+) already uses.
- **Spring Boot 4's own design** keeps both a Jackson 3 `JsonMapper` bean and (when `spring-boot-jackson2` is present) a Jackson 2 `ObjectMapper` bean, and documents a single starter artifact serving both baselines via `@ConditionalOnClass`. This RFC follows that pattern for Wave C.
- **openapi-generator** ships an opt-in `useJackson3` flag for Java 17+ targets; if a spike shows it covers the `native` library and custom templates, it can drive the generated-code namespace switch rather than hand-patching.
- **Existing OpenFGA SDK deprecation idiom:** `ApiClient.urlEncode` already uses `@Deprecated(forRemoval=true, since=…)`, so the bridge follows an established in-repo convention.

## Unresolved Questions
[unresolved-questions]: #unresolved-questions

1. **Feature parity for SB3 / Jackson 2 users.** Accept a 0.x LTS on security-only patches with a single Jackson-3-forward major (recommended), or keep shipping features to Jackson 2 via the adapter-jar architecture? This is the one decision that changes the architecture.
2. **The mapper accessors at the flip.** Remove them entirely (cleanest), or retype to `tools.jackson.*` (still a source break, but a familiar shape)?
3. **The `TypeReference` rider.** SDK-owned type token (recommended), drop the generic overload entirely and keep only `Class<T>` (zero known external callers), or retype to Jackson 3?
4. **Starter versioning.** Single artifact spanning SB3 and SB4 via conditionals (recommended), or a dedicated SB4 major line?
5. **The `useJackson3` generator flag.** Adopt it as the Jackson 3 driver if the spike shows sufficient coverage, or hand-patch templates? (Resolved through implementation.)
6. **`JsonNullable`.** Confirmed zero usages: drop it on the Jackson 3 path, or keep it for forward compatibility?
7. **Out of scope for this RFC:** the specific CI matrix design for the starter (Boot 4 BOM row, JDK version injection) is an implementation detail tracked on PR [#183](https://github.com/openfga/spring-boot-starter/pull/183) and its follow-up issue.
