# RFC: Cross-SDK Conformance Testing for OpenFGA SDKs

## Meta

- **Name:** Cross-SDK Conformance Testing for OpenFGA SDKs
- **Start Date:** 2025-12-28
- **Authors**: @rhamzeh
- **Status:** Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- **RFC Pull Request:** https://github.com/openfga/rfcs/pull/<TBD>
- **Relevant Issues:**
- **Supersedes:** N/A
- **Target repos:** `openfga/{go,js,dotnet,java,python}-sdk`, new `openfga/sdk-conformance`

## Summary

OpenFGA maintains multiple SDKs that must behave consistently across languages. Each SDK currently implements its own manually-authored tests, resulting in duplicated effort, drift, and uneven coverage.

This RFD proposes introducing a shared, versioned conformance test suite authored once in Gherkin and executed in each SDK via language-specific runners. The suite is mock-first and uses WireMock as the canonical backend to provide:

- deterministic API responses,
- sequencing/state (e.g., 401 → refresh → 200),
- fault injection (delays, malformed responses, connection close),
- request introspection (counts, headers, request bodies).

A live OpenFGA smoke suite as an optional integration verification.

## Motivation

### Current pain points

- **Duplicated test logic:** the same behavioral expectations are implemented 5 times.
- **Drift:** differences in retries, header precedence, or error mapping can persist unnoticed.
- **Hard-to-test behaviors:** deterministic retries/failures, malformed responses, timeouts, and token-provider failures are painful with only live API tests.
- **Unstable workflows:** live integration tests are often slow, flaky, and require secrets.
- **Hard to onboard new SDKs:** Onboarding new SDKs requires rewriting new tests from scratch.
- **Lack of visibility on SDK support:** without a shared suite that defines what features each SDK supports.

### Why “API integration tests only” are insufficient

Testing primarily against a live OpenFGA server validates server+transport wiring, but does not allow:

- deterministic retry/failure sequences (503→200, 429→200)
- connection-level faults (reset/truncate)
- precise error payload shaping
- deterministic client-credentials caching and refresh-on-401 behavior
- streaming edge cases (invalid NDJSON, mid-stream error objects, cancellation)

## Goals and Non-Goals

### Goals

- **Define target API version:** Defines the target git hash from the `openfga/api` repo that the conformance tests targets
- **Write once, reuse everywhere:** shared suite represents canonical SDK behavior.
- **Mock-first determinism:** reliable coverage for retries/failures/errors/timeouts.
- **OSS-friendly:** no proprietary tooling; contributor-friendly workflow.
- **Stable workflow:** versioned suite, pinned by SDKs; predictable CI behavior.
- **Coverage parity:** key behaviors from all SDKs are represented; discrepancies are explicit.

### Non-goals

- Removing all per-language unit tests; language-specific unit tests remain.
- Exhaustive validation of OpenFGA server correctness.
- Enforcing identical error class names across languages; validate portable fields/semantics.

## Requirements

### Functional

- Execute shared `.feature` files across Go/JS/.NET/Java/Python SDKs.
- Default suite runs against WireMock (no live dependencies).
- Support request introspection:
  - request count by method+path
  - received headers
  - request bodies (via JSONPath match and/or retrieval)
- Support failure injection:
  - ordered sequences (503→200, 401→200)
  - delays (timeout simulation)
  - malformed JSON / malformed chunks
  - connection close / reset
- Support streaming endpoint testing (NDJSON) where SDK supports it.

### Non-functional

- **Fast and deterministic** for PR CI.
- **No secrets** required for default suite.
- **Stable versioning** for the suite and fixtures.
- **Low maintenance:** minimal step vocabulary + standard stub conventions.

## Proposed Solution

## New repository: `openfga/sdk-conformance`

```t
sdk-conformance/
├── features/                    # Gherkin feature files organized by capability
│   ├── apis/                   # Core API operations (Check, Write, Read, Expand, etc.)
│   │   ├── assertions/         # Assertions API scenarios
│   │   ├── authorization_models/  # Authorization model management
│   │   ├── queries/            # Query operations (Check, Expand, ListObjects, etc.)
│   │   ├── stores/             # Store management operations
│   │   └── tuples/             # Tuple read/write operations
│   ├── authentication/         # Auth flows: bearer token, client credentials, refresh
│   ├── client_behavior/        # SDK client initialization and configuration
│   ├── client_wrappers/        # Convenience methods (ListRelations, ListUsers, etc.)
│   ├── error_scenarios/        # Error handling and edge cases
│   ├── integration/            # End-to-end integration tests
│   ├── observability/          # Observability, logging, telemetry, tracing
│   ├── performance/            # Performance tests and optimization
│   ├── raw_request_executor/   # Raw HTTP request execution tests
│   ├── request_configuration/  # Header, timeout, and request config
│   └── streaming_apis/         # Streaming NDJSON scenarios
├── wiremock/
│   ├── bundles/                # WireMock mapping bundles organized by scenario
│   ├── fixtures/               # JSON request/response fixtures
├── docs/                       # Documentation documentation
├── CHANGELOG.md                # Versioned release history
├── CONTRIBUTING.md             # Contribution guidelines
└── Makefile                    # Build and test automation
```

## WireMock as the canonical mock backend

### Why WireMock

- Mature OSS mock server with:
  - admin APIs for reset, mapping load, and request inspection
  - sequencing/state via "scenarios"
  - common fault injection primitives (delays and connection faults)
- Strong ecosystem: easy Docker-based execution in CI and locally
- Reliable request journaling for assertions and debugging
- Comprehensive mapping language for complex scenarios

### WireMock Implementation

**Docker Integration**: Each SDK should have a docker-compose.yaml file that allows spinning up the wiremock server.

**Mapping Organization**: WireMock stub mappings are organized in `wiremock/bundles/` by scenario domain (e.g., `bundles/apis/queries/check.json`, `bundles/authentication/oauth2.json`). This mirrors the feature file organization, making mappings easy to locate and maintain.

**Fixture Management**: `wiremock/fixtures/` contains canonical JSON request/response fixtures that are referenced by mappings, ensuring consistency across scenarios.

## 5.3 Tagged execution modes

Feature tags enable selective execution based on test scope and environment:

- **`@core`**: Must pass in PR CI; WireMock only; covers basic operations and fundamental behavior; fast and deterministic
- **`@auth`**: Authentication flows including bearer token, client credentials, token refresh; WireMock only
- **`@retry`**: Retry logic with fault injection (429, 503 responses); tests backoff and replay behavior
- **`@headers`**: Header handling: defaults, per-request overrides, header merging
- **`@client`**: Convenience wrapper methods (ListRelations, ListUsers, BatchCheck enhancements)
- **`@streaming`**: NDJSON streaming scenarios including error handling and cancellation
- **`@performance`**: Performance and optimization tests; optional for PR CI
- **`@observability`**: Logging and tracing behavior; optional for PR CI
- **`@integration`**: Live OpenFGA smoke tests; nightly/release only; requires real server
- **Language-specific tags** (e.g., `@go-only`, `@dotnet-only`): Mark scenarios supported by particular SDKs only

## 5.3 Per-Language Runners

Runners should be per-language and located in each repo. It is the responsibility of the runners to interpret the features in the SDK language and call the wiremock server appropriately.

Runners declare which tags they support via configuration. For example:

- A Python runner might skip `@go-only` scenarios
- All runners support `@core` scenarios by definition
- Integration tests (`@integration`) are gated to trusted CI contexts only

## 5.4 Steps

Steps should include:

### Client Configuration Steps

- Basic client setup with store ID and API URL
- Authorization model configuration
- Authentication mode selection (bearer token, client credentials)
- Default header configuration
- Per-request header override

### Authentication Steps

- Bearer token configuration
- Client credentials setup (clientId, clientSecret, issuer, audience, scopes)
- Token acquisition and caching behavior
- 401 refresh-and-replay sequences

### API Method Steps

All core OpenFGA API operations:

- `Check`
- `BatchCheck`
- `Write`
- `Read`
- `Expand`
- `ListObjects`
- `ListUsers`
- `ListStores`
- `ListAssertions`
- `ReadAssertions`
- `WriteAssertions`

### Advanced Feature Steps

- Contextual tuples
- Context objects
- Transaction options in Write
- Streaming
- Pagination
- Correlation IDs

### Request Assertion Steps

- HTTP request count verification
- Header presence and value assertions
- Request body inspection via JSONPath
- Received parameter validation

### Response Assertion Steps

- Successful response validation
- Error response and error code assertions
- Result field value assertions
- Streaming chunk validation

### Portability Principles

- Steps focus on behavior, not language-specific implementation
- Complex data structures use table format for readability
- Async/sync patterns are abstracted; language runners handle implementation
- Sensitive data handling (token values, credentials) is redacted in logs

### Existing Tests Coverage

The conformance suite should incorporate test behaviors from existing SDK implementations. For SDKs such as Python that support both Sync and Async variants, both should be covered, though this would be done at the runners layer.

## Prior Art

TBD

## Unresolved Questions

- How should we surface the supported API version all the way to the SDK Generator to make sure it's generating from the same commit?
