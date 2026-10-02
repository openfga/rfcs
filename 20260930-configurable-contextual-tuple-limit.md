# Meta
[meta]: #meta
- **Name:** Configurable contextual tuple limit and request payload limit
- **Start Date:** 2026-09-30
- **Author(s):** [@tylernix](https://github.com/tylernix)
- **Status:** Draft
- **RFC Pull Request:** (leave blank)
- **Relevant Issues:**
  - https://github.com/openfga/openfga/issues/3021
  - https://github.com/openfga/api/pull/208
  - https://github.com/openfga/cli/issues/564
- **Supersedes:** N/A

# Summary

The number of contextual tuples per request is hardcoded to 100. This RFC makes it an optional, server-level configuration flag (`--max-contextual-tuples`, default 100). To protect the server, a request payload size limitation should be implemented. The request payload limit is validated first, so it is enforced first whenever both are exceeded.

# Definitions

- **Contextual tuples:** tuples sent with a request (Check, BatchCheck, ListObjects, StreamedListObjects, ListUsers, Expand) instead of written to the store beforehand. https://openfga.dev/docs/interacting/contextual-tuples

# Motivation

- Over the years, the contextual tuple limit has moved from 10, 20, then [100](https://github.com/openfga/api/pull/208).
- Some adopters pass every identity group as contextual tuples and easily exceed 100 ([#3021](https://github.com/openfga/openfga/issues/3021)).
- The CLI turns authorization model test fixtures (`.fga.yaml`) into contextual tuples, so large test suites hit the same cap.

# Proposal

Two enforced limits, evaluated in order:

| Order | Limit | Flag | Default | On breach |
|:--|:--|:--|:--|:--|
| 1 | Payload size (bytes) | `--grpc-max-recv-msg-bytes` already exists, but should also be applied to HTTP | 616,448 | HTTP 413 / gRPC `ResourceExhausted` |
| 2 | Contextual tuples per request | `--max-contextual-tuples` (new) | 100 | `InvalidArgument` (HTTP 400), same as today |

Payload size limit is the ceiling. The contextual tuple count is a configurable guardrail beneath it. Raising the count past what fits in the payload limit has no effect. A typical 100-tuple request is well under 616,448 bytes, so the count fires first.

The count applies per request, and per check item in BatchCheck.

# How it works today

Today the 100 contextual tuple limit lives in generated validation in `openfga/api`: [`ContextualTupleKeys.tuple_keys`](https://github.com/openfga/api/blob/c0650ce2169bcb39a72d73841fb531c200e32248/openfga/v1/openfga.proto#L218-L224) and [`ListUsersRequest.contextual_tuples`](https://github.com/openfga/api/blob/c0650ce2169bcb39a72d73841fb531c200e32248/openfga/v1/openfga_service.proto#L1095-L1099), both `max_items = 100`.

The request payload limit already exists on the gRPC server: the [`--grpc-max-recv-msg-bytes`](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/cmd/run/run.go#L172) flag feeds `grpc.MaxRecvMsgSize`. The default is `DefaultMaxRPCMessageSizeInBytes`, and [`.config-schema.json`](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/.config-schema.json#L373-L378) documents it. The gateway only hits the gRPC limit after it has decoded the body. 

A new HTTP middleware rejects on `Content-Length` and wraps the body with `http.MaxBytesReader` for chunked requests, reusing the same value.

# Work breakdown

| # | Chunk | Repo | Depends on |
|:--|:--|:--|:--|
| 1 | Max contextual tuples flag (config only) | openfga/openfga | none |
| 2 | Enforce at the API endpoints | openfga/api, openfga/openfga | 1 |
| 3 | Request body size limiting | openfga/openfga | none |
| 4 | CLI and model test harness | openfga/cli | 1, 2 |
| 5 | Documentation | openfga/openfga.dev | 1 to 4 |

## 1. Server-Level Configuration Flag (good first issue)
- **Today:** the 100 contextual tuple limit lives in generated validation in `openfga/api`: [`ContextualTupleKeys.tuple_keys`](https://github.com/openfga/api/blob/c0650ce2169bcb39a72d73841fb531c200e32248/openfga/v1/openfga.proto#L218-L224) and [`ListUsersRequest.contextual_tuples`](https://github.com/openfga/api/blob/c0650ce2169bcb39a72d73841fb531c200e32248/openfga/v1/openfga_service.proto#L1095-L1099), both `max_items = 100`.
- **Change:** add `MaxContextualTuples` (default 100) to [`config.go`](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/pkg/server/config/config.go) and [`run.go`](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/cmd/run/run.go). 

Can use `max-checks-per-batch-check`[1](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/pkg/server/config/config.go#L402-L404)[2](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/cmd/run/run.go#L278) as an example. 

- Flag `--max-contextual-tuples`, variable `OPENFGA_MAX_CONTEXTUAL_TUPLES`, key `maxContextualTuples`.
- Entry in `.config-schema.json`; config validation rejects values `<= 0`.

`server.WithMaxContextualTuples(n)` stores the value, but not yet enforced until Chunk 2 work is complete.

## 2. Enforce Limits at the API Endpoints
- **Today:** the proto rejects more than 100 contextual tuples before the server sees them ([`ContextualTupleKeys.tuple_keys`](https://github.com/openfga/api/blob/c0650ce2169bcb39a72d73841fb531c200e32248/openfga/v1/openfga.proto#L218-L224)).
- **Change:** remove that cap in `openfga/api`, then check `MaxContextualTuples` in each handler, starting with the [`Check` handler](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/pkg/server/check.go#L52-L56).

Can use the [`MaxTuplesPerWrite` check](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/pkg/server/commands/write.go#L211-L213) as an example of a count limit.

- In `openfga/api`: delete `max_items = 100` from `ContextualTupleKeys.tuple_keys` and `ListUsersRequest.contextual_tuples`. Leave `Assertion.contextual_tuples` (20) alone.
- In `openfga/openfga`: bump `openfga/api` in `go.mod`, then add the count check to Check, BatchCheck (per item), Expand, ListObjects, StreamedListObjects, and ListUsers.
- Check in the handler, not the validator interceptor. The CLI's built-in test server calls handlers directly and skips interceptors.
- Put the check next to the `if !validator.RequestIsValidatedFromContext` block, not inside it. On a normal server that block is skipped.
- Return `InvalidArgument` and name the limit in the message.
- Tests per endpoint: at the limit, over the limit, limit raised above 100, and payload breach wins over count breach.

Once this lands, the Chunk 1 flag takes effect. Needs Chunk 1.

## 3. Request Body Size Limit (good first issue)
- **Today:** only gRPC has a request size limit, set by [`grpc.MaxRecvMsgSize`](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/cmd/run/run.go#L579). HTTP requests have no limit yet.
- **Change:** add a size check to the HTTP server in [`runHTTPServer`](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/cmd/run/run.go#L797-L799).

Can use [`HTTPPanicRecoveryHandler`](https://github.com/openfga/openfga/blob/97943bf64d85ac3015272d90ab1145696db9ef5b/pkg/middleware/recovery/recovery.go#L22) as an example of HTTP middleware.

- Add a middleware that wraps the handler chain.
- Reject with 413 when `Content-Length` is over the limit. Otherwise wrap the body with `http.MaxBytesReader`.
- Reuse `grpc.maxRecvMsgBytes`. No new flag.
- Chunked requests have no `Content-Length`, and the gateway turns the read error into a 400. Match `http: request body too large` in the gateway error handler to return 413 instead.
- Tests: add an HTTP version of the oversize test in `tests/functional_test.go`. Cover over the limit, under the limit, and a chunked body.

Does not depend on the other chunks, so it can start right away.

## 4. CLI and Model Test Harness Flag
- **Today:** `fga model test` runs tests on a built-in OpenFGA server. Only [`--max-types-per-authorization-model`](https://github.com/openfga/cli/blob/80e2edfe32693363101a795de459a5165109980e/cmd/model/test.go#L70-L82) is passed to it.
- **Change:** add `MaxContextualTuples` to [`LocalServerConfig`](https://github.com/openfga/cli/blob/80e2edfe32693363101a795de459a5165109980e/internal/storetest/tests.go#L18-L23) and a matching `--max-contextual-tuples` flag.

Can use `--max-types-per-authorization-model` as an example.

- Flag `--max-contextual-tuples` on `fga model test`, default 100.
- Pass it to the built-in server with `server.WithMaxContextualTuples`.
- Needs a release of `openfga/openfga` that has the option. The CLI pins v1.21.0 today.
- Remote runs (`--store-id`) ignore the flag. The server's own limit applies.
- `fga query` needs no change. It always talks to a server.
- Optional follow-up: add a `fga_cli_max_contextual_tuples` input to `action-openfga-test`.

Local tests call the server directly, so only the tuple count applies there, not the payload limit. Needs Chunks 1 and 2.

## 5. Documentation
- **Today:** the [contextual tuples page](https://github.com/openfga/openfga.dev/blob/fb5359796e1e1027985675cfdd2413e6dcbfdf5d/docs/content/interacting/contextual-tuples.mdx#L33) says "currently a limit of 100".
- **Change:** document the new flag in the [configuration reference](https://github.com/openfga/openfga.dev/blob/fb5359796e1e1027985675cfdd2413e6dcbfdf5d/docs/content/getting-started/setup-openfga/configuration.mdx).

Can use the [`maxChecksPerBatchCheck` row](https://github.com/openfga/openfga.dev/blob/fb5359796e1e1027985675cfdd2413e6dcbfdf5d/docs/content/getting-started/setup-openfga/configuration.mdx#L115) as an example.

- Add a `maxContextualTuples` row: env var `OPENFGA_MAX_CONTEXTUAL_TUPLES`, flag `max-contextual-tuples`, default `100`.
- Update the `grpc.maxRecvMsgBytes` row to say it also covers HTTP.
- Contextual tuples page: replace the "limit of 100" line. Explain both limits, the order, and show the error for each.
- CLI docs: add `--max-contextual-tuples` to `fga model test`.
- Add CHANGELOG entries in each repo.

Last chunk. Write it after Chunks 1 to 4 ship, so the docs match real behavior.

# Migration

No breaking changes at defaults. Count stays 100, payload stays 616,448 bytes.

Release order: `openfga/api`, then `openfga/openfga`, then `openfga/cli`.

# Drawbacks

- One more config option to document and maintain.
- Limits differ per deployment. A test passing locally with a raised limit can fail against a stricter server.
- Removing the proto cap removes a guardrail from anyone relying on generated validators alone. The server becomes the single enforcer.