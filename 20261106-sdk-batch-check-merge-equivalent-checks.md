# Meta
[meta]: #meta
- **Name:** Merge equivalent checks in SDK BatchCheck
- **Start Date:** 2026-11-06
- **Author(s):** [@Siddhant-K-code](https://github.com/Siddhant-K-code)
- **Status:** Approved
- **RFC Pull Request:** https://github.com/openfga/rfcs/pull/40
- **Relevant Issues:**
  - https://github.com/openfga/python-sdk/issues/325
- **Supersedes:** N/A

# Summary
[summary]: #summary

A new opt-in BatchCheck option, `merge_equivalent_checks`, for the OpenFGA SDKs. When it's on, the SDK finds checks that are certain to get the same answer, sends each of them once, and copies the answer back to every original check. In this first phase, two cases qualify:

- **Aliases:** relations defined as just another relation, like `define can_share: can_view`.
- **Exact repeats:** the same check passed more than once.

The part that reads the model (Resolve) lives in `openfga/language` (Go first), with shared test cases that every SDK must pass. The SDK caches the model, so it's read once per model ID, and reports how many checks it actually sent. The Python SDK ships it first, as a beta. Later phases are listed in [Phases](#phases).

# Definitions
[definitions]: #definitions

- **Alias:** a relation defined as exactly one other relation on the same object, and nothing else. Example: `define can_share: can_view`.
- **Alias chain:** an alias that points to another alias. Example: `can_share → can_view → viewer`.
- **Final relation:** where a chain ends, at the first relation that isn't an alias. It can be defined in any way, including with `or`, `and`, `but not` or `from`.
- **Group:** checks with the same final relation, user, object, context and contextual tuples. The SDK sends one check per group.
- **Userset user:** a user written as `type:id#relation`, like `team:eng#member`.

# Motivation
[motivation]: #motivation

- We recommend fine-grained permissions such as `can_share` and `can_view_full_name`, instead of checking one broad relation everywhere. Then the model can change later without changing app code.
- Apps that follow this advice often check several of these permissions for the same user and object at once. Many are aliases, so their answers are always the same.
- Where rate limits or usage are counted per check (for example on a managed service, or behind an API gateway), each of those checks still counts. Following our advice costs more.
- The server can't fix this. Those limits apply to the incoming request, before the server reads the model. The repeated work has to be removed before the request is sent.
- **Expected outcome:** fewer checks sent, the same answers, and no change for anyone who doesn't turn the option on.

# What it is
[what-it-is]: #what-it-is

**For application developers using an OpenFGA SDK:**

- A new BatchCheck option, `merge_equivalent_checks`, off by default. Each SDK uses its own casing (see [The option in each SDK](#the-option-in-each-sdk)).
- It needs an authorization model ID, either set on the client or passed in the call. Without one, the SDK raises an error before sending anything. The SDK reads that model once and caches it.
- You still get one result for each check you passed in, matched by your correlation IDs.
- The SDK reports how many checks it actually sent, in [telemetry](#telemetry) and, in Python, in the response (see [Per-call count](#per-call-count)).
- Nothing else about calling BatchCheck changes.

## Example 1: three checks become one

```dsl
model
  schema 1.1
type user
type document
  relations
    define viewer: [user]
    define can_view: viewer
    define can_share: can_view
    define can_view_full_name: can_view
```

```python
checks = [
    ClientBatchCheckItem(user="user:anne", relation="can_view", object="document:roadmap", correlation_id="view"),
    ClientBatchCheckItem(user="user:anne", relation="can_share", object="document:roadmap", correlation_id="share"),
    ClientBatchCheckItem(user="user:anne", relation="can_view_full_name", object="document:roadmap", correlation_id="full-name"),
]
response = await fga_client.batch_check(
    ClientBatchCheckRequest(checks=checks),
    {"merge_equivalent_checks": True},
)
```

All three relations end at `viewer`. The SDK sends **1 check** (`user:anne`, `viewer`, `document:roadmap`) instead of 3. The response still has 3 results (`view`, `share` and `full-name`), each with the answer from that one check, and `response.checks_sent` is `1`.

## Example 2: what merges and what doesn't

```dsl
model
  schema 1.1
type user
type team
  relations
    define member: [user]
type document
  relations
    define viewer: [user, team#member]
    define editor: [user]
    define can_edit: editor
    define can_view: can_edit or viewer
    define can_share: can_view
```

All checks are on `document:1`.

| # | User | Relation | Ends at | What the SDK sends |
|---|---|---|---|---|
| 1 | `user:anne` | `can_view` | `can_view` (uses `or`, so the chain stops) | one `can_view` check for #1 and #2 |
| 2 | `user:anne` | `can_share` | `can_view` | merged into #1 |
| 3 | `user:anne` | `can_edit` | `editor` | `editor` (nothing to merge with) |
| 4 | `user:bob` | `can_share` | `can_view` | `can_view` (different user) |
| 5 | `team:eng#member` | `can_share` | `can_view` | `can_view` (different user) |

5 checks, 4 sent. Every check goes out with its final relation, even when nothing merges with it (#3, #4, #5). `can_view` includes `can_edit`, but #1 and #3 end at different relations, so they aren't merged.

## The option in each SDK

The same option in every SDK, so later implementations match `language` and Python:

| SDK | How you turn it on | Type and default |
|---|---|---|
| Python | `batch_check(body, {"merge_equivalent_checks": True})` | `bool`, `False` |
| Go | `BatchCheckOptions{MergeEquivalentChecks: openfga.PtrBool(true)}` | `*bool`, `nil` means off |
| JS | `batchCheck(body, { mergeEquivalentChecks: true })` | `boolean`, `false` |
| Java | `new ClientBatchCheckOptions().mergeEquivalentChecks(true)` | `Boolean`, `false` |

Everything else is the same in every SDK: the merge rules, errors, telemetry names and shared tests. For now, only Python adds a [per-call count](#per-call-count) to the response. Other SDKs (like .NET) follow the same pattern.

# How it Works
[how-it-works]: #how-it-works

## Flow

When the option is on, the SDK:

1. Fixes the store ID and model ID for the whole call. Reading the model and sending the checks must use the same model, or merges could be wrong.
2. Validates every correlation ID you passed in (see [Errors](#errors)), so merging can't hide an invalid one.
3. Gets the model from the [model cache](#model-cache). Only the first call for a model reads it from the server, and calls running in parallel share that one read. If the model can't be read, the SDK skips steps 4 and 6 and sends the checks unmerged, as it does today (see [Errors](#errors)).
4. **Resolve:** groups the checks and builds the list to send.
5. Sends that list in batches, as it does today. Grouping covers all checks in the call before they're split into batches, so checks that would land in different batches can still merge.
6. **Expand:** copies each answer back to every check in its group.
7. Records [telemetry](#telemetry).

In Example 1, resolve makes one group (`user:anne`, `viewer`, `document:roadmap`) holding `view`, `share` and `full-name`. The SDK sends one check, and expand copies its answer to all three.

## Merge rules

1. **What counts as an alias:** only a relation written as exactly one other relation, `define a: b` (in JSON, a lone `computedUserset`). Nothing else is an alias: not direct types (`[user]`), `or`, `and`, `but not`, `from`, or a union with a single child.
2. **Chains:** follow aliases until you reach a relation that isn't one. If the chain points to a relation that doesn't exist, or loops, don't merge it. A valid model shouldn't contain these, so this is only a safety net.
3. **Groups:** checks merge only if the final relation, user, object, context and contextual tuples all match. This also merges exact repeats. The sent check uses the context and contextual tuples of the group's first check.
4. **"Match" means the server would get exactly the same values.** When unsure, don't merge. Sending a check unmerged is always safe.
   - Context must match exactly. A missing or extra field means no match, and strings must match byte for byte (one extra space is a different value). No context doesn't match an empty context (`{}`), in a check or in a contextual tuple's condition.
   - Numbers match only if they're the same double, including the sign: `1` and `1.0` match, `0` and `-0.0` don't (a condition can tell them apart).
   - Types matter: `true` and `1` don't match.
   - Key order in objects doesn't matter.
   - Order and repeats in lists do matter.
   - Contextual tuples can be in any order. Each tuple needs the exact same user, relation, object, condition name and condition context, or no condition on both. No contextual tuples doesn't match an empty list (`[]`).
   - These rules apply at every level of nested objects and lists.
5. **Sent as is, with the caller's relation and correlation ID, and never merged (not even with an exact repeat):**
   - **A userset on the same object, whose relation is anywhere in the chain** (including the one you asked for). With Example 1's model, take `document:roadmap#can_view` checking `can_share` on `document:roadmap`. The server's older check algorithm says a userset always has its own relation on its own object, and it applies this at every step of a chain. So `can_share → can_view` hits that rule and returns true, while the merged `viewer` check would be answered from tuples. The newer weighted-graph algorithm doesn't have this rule. This is the only special userset case: other usersets, like `team:eng#member`, merge like any user, and so do wildcard users (`user:*`).
   - **Contextual tuples that repeat a user, relation and object**, with the same condition or different ones. The server's answer can depend on which one it reads first, so the SDK sends these checks exactly as written. This is rare, and sending as is is always safe.
   - **Anything not in the model:** an unknown object type, relation, user type or userset relation (like `team:eng#unknown`). The server returns its usual error, so it stays the single source of these errors.
6. **Every group is sent with its final relation**, a group of one too. For example, a lone `can_share` check goes out as `can_view`. The server then sees fewer distinct relations, so it can cache better. Checks sent as is (rule 5) keep the caller's relation.
7. **Correlation IDs:**
   - A group of one keeps the caller's ID.
   - A merged group gets an ID generated by the SDK. The sent check stands for the whole group, so reusing one caller's ID would make its answer or error look like it belongs to that one check.
   - Only merged groups are numbered, in the order of their first check (`merged-1`, `merged-2`, ...). The IDs follow the server's ID rule and never match a caller's ID in the same call. If one would, the SDK skips to the next free number.
8. **Deterministic from the start:** the same model and the same checks, in the same order, always give the same groups, the same checks to send in the same order, and the same IDs. Each group is sent at the position of its first check.

## Expand rules

- A group's answer (allowed, or an error) is copied to every check in the group.
- If the server returns no answer for a group, every check in it gets an error. A missing answer is never treated as "not allowed".
- Answers with unknown correlation IDs are ignored.

## Errors

These apply only when the option is on. With it off, nothing changes.

| When | What happens |
|---|---|
| No model ID | Validation error, before anything is sent. It never falls back: the mistake is in the caller's setup, so falling back would mean the option silently never works. |
| A correlation ID you passed in is invalid | Validation error, before anything is sent. The rule is `^[\w\d-]{1,36}$` (ASCII only), and it's part of the API contract in `openfga/api`. Without this check, merging could hide an invalid ID that the server would have rejected. |
| Duplicate correlation IDs | Validation error, as today. |
| Reading the model fails (for example 401, 403, 404, 429, or a network error) | The SDK sends the checks unmerged, as it does today, so turning the option on can't break a call that works now. It logs one warning per model (not per call), records the fallback in [telemetry](#telemetry), and waits a short time (for example, a minute) before trying to read that model again. On a 404, the BatchCheck itself then gets the server's usual error. |
| A check's object type, relation, user type or userset relation isn't in the model | Sent as is. The server returns its usual error. |
| A BatchCheck request fails | The call fails, as today. |
| One check in a request returns an error | Only that check's group is affected. The error is copied to every check in the group. For example, if `can_share` was merged into `can_view` and `can_view` returns an error, `can_share` gets the same error. |

## Model cache

Reading the model on every call would add a ReadAuthorizationModel call to each BatchCheck. That endpoint often has much lower rate limits than Check, so the SDK caches models:

- In memory, per client instance, keyed by store ID and model ID.
- A model never changes once it's written, so a cached model never goes stale.
- Only successful reads are cached. Calls for the same model share one in-flight read, so 50 parallel BatchCheck calls make one ReadAuthorizationModel request.
- A failed read is remembered for a short time, so missing read access doesn't add a read to every call.
- It holds a small, bounded number of models and drops the least recently used.
- To benefit, create the client once and reuse it, as the SDKs already recommend.

## Where the logic lives

- **Resolve** lives in `openfga/language`, starting with Go. It takes the model and the checks, and returns the checks to send plus the groups.
- **Expand** lives in each SDK. It takes the groups and the server's answers, and returns one answer per original check.

Why split it this way:

| | Pros | Cons |
|---|---|---|
| **Resolve in `language`** | It reads the model, and `language` already parses and validates models. The rules and tests are written once, and if the way models are read changes, it changes there first. The server, CLI or other tools could reuse it later. | SDKs that can't call `language` (like Python today) have to port it. The shared tests keep ports in line. |
| **Expand in each SDK** | It doesn't read the model. It only maps answers back by correlation ID, using each SDK's own response and error types, so there's nothing model-specific to share. | Each SDK writes it. It's short, and each SDK tests it against the [Expand rules](#expand-rules). |

SDKs that can call `language` use Resolve directly. The rest port it. Either way, they must pass the shared tests.

## Shared tests

- A new file in `openfga/language`, `tests/data/equivalent-checks-cases.yaml`, in the same style as the existing test-case files. A generated copy, `equivalent-checks-cases.json`, has every model as JSON for SDKs without a DSL parser, and a Go test keeps the two in sync.
- Each case has a model, the input checks, and the expected checks to send, each with its ID and the original checks it stands for.
- Cases must cover the tricky matches above: `1` vs `1.0`, integers that are the same double, `0` vs `-0.0`, `true` vs `1`, missing and extra fields, no context vs `{}`, no contextual tuples vs `[]`, key order, list order, contextual tuple order, repeated contextual tuples and nested values. Also the same-object userset case, checks not in the model, and generated IDs that would match a caller's ID.
- **How SDKs use it:** today, the Go, JS and Java code in `language` reads `tests/data/*.yaml` straight from the repo, and no SDK uses these files yet. For SDKs this is new: a small script copies the cases from a pinned `language` commit into the SDK repo, and CI runs them. Bumping the pin is a manual PR for now. Automating it is a [later phase](#phases).

A case looks like this (all cases are in [openfga/language#718](https://github.com/openfga/language/pull/718)):

```yaml
- name: alias chain merges into the final relation
  dsl: |
    model
      schema 1.1
    type user
    type document
      relations
        define viewer: [user]
        define can_view: viewer
        define can_share: can_view
  checks:
    - { id: a, user: "user:anne", relation: can_view, object: "document:1" }
    - { id: b, user: "user:anne", relation: can_share, object: "document:1" }
    - { id: c, user: "user:bob", relation: can_share, object: "document:1" }
  expected_sent:
    - { id: merged-1, user: "user:anne", relation: viewer, object: "document:1", for: [a, b] }
    - { id: c, user: "user:bob", relation: viewer, object: "document:1", for: [c] }
```

## Telemetry

Metrics, recorded for each BatchCheck call with the option on. Names follow the existing `fga-client.*` metrics and will be settled during implementation.

| Metric | Type | What it records |
|---|---|---|
| `fga-client.batch_check.checks_requested` | counter | Checks you passed in. |
| `fga-client.batch_check.checks_sent` | counter | Checks actually sent. The gap between the two is what the option saved. |
| `fga-client.model_cache.lookup` | counter | Model cache lookups, with a `hit` or `miss` attribute. |
| `fga-client.batch_check.merge_fallback` | counter | Calls sent unmerged because the model couldn't be read, with the status code. |

- Attributes are store ID and model ID, like the existing metrics. No users, objects or context.
- The existing `fga-client.request.batch_check_size` attribute stays. It only reports how many checks each request sent, not how many you passed in.
- On a fallback, the SDK logs one warning per model, with the store ID, model ID and status code.
- Metrics add up across calls, so they can't show what one call sent. Python's response shows that (see [Per-call count](#per-call-count)).
- Resolve is deterministic, so any result can be reproduced from the model and the original checks.

## Per-call count

- Python's `ClientBatchCheckResponse` gets an optional `checks_sent`: how many checks this call actually sent. It's set only when the option is on (including when the SDK falls back to sending unmerged), so nothing changes with it off.
- It's for apps that limit requests per client on their side, which metrics can't show, since they add up across calls.
- Other SDKs decide when they add the option. JS and Java return their own response type, so a field is additive. Go's `BatchCheckExecute` returns the generated `BatchCheckResponse`, so Go would need a new return type or a change to the generated model.

# Phases
[phases]: #phases

**Phase 0 (this RFC):**

- BatchCheck only, in the Python SDK, as a beta.
- Aliases (including chains) and exact repeats.
- Resolve in Go in `language`, with the shared tests.
- The model cache, telemetry and Python's per-call count.

**Later phases**, each with its own review. Resolve already returns groups keyed by a final relation, so none of these change the option or the shape of Resolve's output.

- **Equivalent relations:** relations that aren't aliases but always get the same answer. For example:
  - The same definition: `define can_edit: editor or owner or editor from parent` and `define can_share: editor or owner or editor from parent`.
  - A union where every branch ends at one relation: with `define can_edit: editor`, `define can_view: can_edit or editor` is the same as `editor`.
- **Answering from the model:** skip checks the model shows can never be true. For example, in Example 2, `team:eng#member` can never have `can_edit`, because `editor` only allows `user`. Also return errors from the SDK for checks not in the model.
- **More methods:** ClientBatchCheck, ListRelations (which uses ClientBatchCheck), and ReadAuthorizationModel using the model cache.
- **More SDKs:** JS and Java versions of Resolve in `language`, then the other SDKs, using [the same option](#the-option-in-each-sdk). Each one decides whether to add the [per-call count](#per-call-count).
- **Automation:** open a PR in each SDK when Resolve or the shared tests change in `language`.

# Migration
[migration]: #migration

None. The option is off by default. Python's response gains one optional field, set only when the option is on. The Python SDK ships the option marked **beta**, so its behavior may change based on feedback.

# Drawbacks
[drawbacks]: #drawbacks

- **One model read per model ID:** the first BatchCheck for each model in a client reads it. Short-lived clients don't benefit from the cache. For small batches with few aliases, the option may save little or nothing. If the credentials can't read the model, it saves nothing; the warning and the fallback metric make that visible.
- **Harder to debug:** the SDK changes what it sends. Every check goes out with its final relation, so a server error names `viewer`, not the `can_share` you asked for.
- **Risk of drift:** a ported Resolve could read the model differently from `language` or the server. This is reduced by keeping the rules in `language`, running the shared tests, and later opening a PR in each SDK automatically when `language` changes.

# Alternatives
[alternatives]: #alternatives

- **Merge on the server:** this saves server work but doesn't reduce counted checks, because limits apply to the request before the server reads the model. It's still worth doing separately.
- **Each SDK writes its own rules:** they would drift apart and could disagree with the server. Keeping the rules in `language` with shared tests avoids that.
- **A separate wrapper method** (like `batch_check_merged`) instead of an option: every entry point (BatchCheck, ClientBatchCheck, ListRelations) would need its own parallel method, and apps would have to change their call sites. An option keeps one method per entry point and extends to the others in later phases.
- **Fail when the model can't be read**, maybe with a setting to fall back: the option is a cost saving, so it shouldn't break a call that works today. Falling back with a warning and a metric keeps the call working, still tells the developer, and avoids another setting.
- **Return errors from the SDK for checks not in the model:** this saves a check, but the SDK would own those errors and have to keep them in line with the server. It's listed as a later phase.
- **A standalone code snippet for developers:** no one maintains it, and every app re-implements subtle rules.
- **Do nothing:** apps keep choosing between fine-grained permissions and fewer counted checks.

# Prior Art
[prior-art]: #prior-art

- The server already merges exact repeats within one BatchCheck (OpenFGA v1.8.1, [openfga/openfga#2102](https://github.com/openfga/openfga/pull/2102)). It doesn't merge aliases, and it runs after the request has been counted.
- The [Queries with Usersets](https://github.com/openfga/rfcs/blob/main/20240328-queries-with-usersets.md) RFC covers how the server answers checks for userset users.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

**To settle before merging:**

1. **Function names and package** in `language`.
2. **Beta release:** a pre-release version (e.g. `0.11.0b1`), or a normal release with the option marked beta?
