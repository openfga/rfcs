# Meta
[meta]: #meta
- **Name:** Merge equivalent checks in SDK BatchCheck
- **Start Date:** 2026-11-06
- **Author(s):** [@Siddhant-K-code](https://github.com/Siddhant-K-code)
- **Status:** Draft
- **RFC Pull Request:**
- **Relevant Issues:**
  - https://github.com/openfga/python-sdk/issues/325
- **Supersedes:** N/A

# Summary
[summary]: #summary

A new opt-in BatchCheck option for the OpenFGA SDKs. When it's on, the SDK finds checks that are certain to get the same answer, sends each of them once, and copies the answer back to every original check. Two cases qualify:

- **Aliases:** relations defined as just another relation, like `define can_share: can_view`.
- **Exact repeats:** the same check passed more than once.

The merge rules live in `openfga/language` (Go first), with shared test cases that every SDK must pass. The SDK caches the model, so it's read once per model ID, and the response reports how many checks were actually sent. The Python SDK ships it first, as a beta.

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

- A new BatchCheck option, off by default. The name is still open (see [Unresolved Questions](#unresolved-questions)).
- It needs an authorization model ID, either set on the client or passed in the call. Without one, the SDK raises an error before sending anything. The SDK reads that model once and caches it.
- You still get one result for each check you passed in, matched by your correlation IDs.
- The response also says how many checks were actually sent (`checks_sent`, name to be decided), so you can track usage against your own limits. It's always set. With the option off, it equals the number of checks you passed in.
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
    {"merge_equivalent_checks": True},  # option name to be decided
)
```

All three relations end at `viewer`. The SDK sends **1 check** (`user:anne`, `viewer`, `document:roadmap`) instead of 3. The response still has 3 results (`view`, `share` and `full-name`), each with the answer from that one check, and `checks_sent` is 1.

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

| # | User | Relation | Ends at | What the SDK sends |
|---|---|---|---|---|
| 1 | `user:anne` | `can_view` | `can_view` (uses `or`, so the chain stops) | one `can_view` check for #1 and #2 |
| 2 | `user:anne` | `can_share` | `can_view` | merged into #1 |
| 3 | `user:anne` | `can_edit` | `editor` | sent as is (nothing to merge with) |
| 4 | `user:bob` | `can_share` | `can_view` | sent as is (different user) |
| 5 | `team:eng#member` | `can_share` | n/a | sent as is (userset users are never merged) |

5 checks, 4 sent. `can_view` includes `can_edit`, but #1 and #3 end at different relations, so they aren't merged. Using that kind of overlap is out of scope.

# How it Works
[how-it-works]: #how-it-works

## Flow

When the option is on, the SDK:

1. Fixes the store ID and model ID for the whole call. Reading the model and sending the checks must use the same model, or merges could be wrong.
2. Validates every correlation ID you passed in (see [Errors](#errors)), so merging can't hide an invalid one.
3. Gets the model from the [model cache](#model-cache), reading it from the server only the first time.
4. **Resolve:** groups the checks and builds the list to send.
5. Sends that list in batches, as it does today. Grouping covers all checks in the call before they're split into batches, so checks that would land in different batches can still merge.
6. **Expand:** copies each answer back to every check in its group.

In Example 1, resolve makes one group (`user:anne`, `viewer`, `document:roadmap`) holding `view`, `share` and `full-name`. The SDK sends one check, and expand copies its answer to all three.

## Merge rules

1. **What counts as an alias:** only a relation written as exactly one other relation, `define a: b` (in JSON, a lone `computedUserset`). Nothing else is an alias: not direct types (`[user]`), `or`, `and`, `but not`, `from`, or a union with a single child.
2. **Chains:** follow aliases until you reach a relation that isn't one. If the chain points to a relation that doesn't exist, or loops, don't merge it. A valid model shouldn't contain these, so this is only a safety net.
3. **Groups:** checks merge only if the final relation, user, object, context and contextual tuples all match. This also merges exact repeats.
4. **"Match" means the server would get exactly the same values.** When unsure, don't merge. Sending a check unmerged is always safe.
   - Numbers match only if they're the same double, including the sign: `1` and `1.0` match, `0` and `-0.0` don't (a condition can tell them apart).
   - Types matter: `true` and `1` don't match.
   - Key order in objects doesn't matter.
   - Order and repeats in lists inside context do matter.
   - Contextual tuples can be in any order (the server ignores their order too), but must otherwise be the same, including any condition context on them. Repeats still count.
   - These rules apply at every level of nested objects and lists.
5. **Always sent as is, never merged:**
   - **Userset users** (like `team:eng#member`). For these, an alias and its final relation can get different answers, and the answer also differs between the server's check algorithms. For example, asking whether `document:1#can_edit` has `can_edit` on `document:1` is true on the older algorithm but false on the newer weighted-graph one.
   - **Wildcard users** (`user:*`). Checks for them are rare, so they're left out to keep the rules simple.
   - **Anything not in the model:** an unknown object type, relation or user type. The server returns its usual error.
6. **A group with one check** is sent as is, with the caller's relation and correlation ID.
7. **A merged group** is sent with the final relation and a new correlation ID generated by the SDK. It never reuses one of the caller's IDs.

## Expand rules

- A group's answer (allowed, or an error) is copied to every check in the group.
- If the server returns no answer for a group, every check in it gets an error. A missing answer is never treated as "not allowed".
- Answers with unknown correlation IDs are ignored.

## Errors

| When | What happens |
|---|---|
| No model ID | Validation error, before anything is sent. |
| A correlation ID you passed in is invalid (the server's rule is `^[\w\d-]{1,36}$`, ASCII only) | Validation error, before anything is sent. Without this, merging could replace an invalid ID that the server would have rejected. |
| Duplicate correlation IDs | Validation error, as today. |
| Reading the model fails | The call fails. The SDK doesn't fall back to sending the checks unmerged. |
| A BatchCheck request fails | The call fails, as today. |
| One check in a request returns an error | Only that check's group is affected. The error is copied to every check in the group. For example, if `can_share` was merged into `can_view` and `can_view` returns an error, `can_share` gets the same error. |

## Model cache

Reading the model on every call would add a ReadAuthorizationModel call to each BatchCheck. That endpoint often has much lower rate limits than Check, so the SDK caches models:

- In memory, per client instance, keyed by store ID and model ID.
- A model never changes once it's written, so a cached model never goes stale.
- Only successful reads are cached. Concurrent calls for the same model share one read.
- It holds a small, bounded number of models and drops the least recently used.
- To benefit, create the client once and reuse it, as the SDKs already recommend.

## Where the logic lives

There are two functions in `openfga/language`, starting with Go:

- **Resolve:** takes the model and the checks, and returns the checks to send plus the groups.
- **Expand:** takes the groups and the server's answers, and returns one answer per original check.

`language` already parses and validates models, so the rules are written and tested once instead of in every SDK. If the way models are read ever changes, it changes there first.

SDKs that can call `language` use it directly. The rest (like Python today) port the two functions. Either way, they must pass the shared tests.

## Shared tests

- A new file in `openfga/language`, `tests/data/equivalent-checks-cases.yaml`, in the same style as the existing test-case files.
- Each case has a model, the input checks, and the expected groups.
- Cases must cover the tricky matches above: `1` vs `1.0`, `0` vs `-0.0`, `true` vs `1`, key order, list order, contextual tuple order, and nested values.
- Each SDK copies the file from a pinned `language` commit and runs it in CI.

A case could look like this (the final format will be settled during implementation):

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
  expected_groups:
    - { user: "user:anne", relation: viewer, object: "document:1", ids: [a, b] }
```

## Observability

- No new logs that include users, objects or context.
- `checks_sent` in the response gives the total for the call.
- The existing `fga-client.request.batch_check_size` telemetry attribute already reports how many checks each request sent.
- Resolve always gives the same output for the same input, so any result can be reproduced from the model and the original checks.

# Migration
[migration]: #migration

None. The option is off by default, and `checks_sent` is a new response field that doesn't change existing ones. The Python SDK ships the option marked **beta**, so its behavior may change based on feedback.

# Drawbacks
[drawbacks]: #drawbacks

- **One model read per model ID:** the first BatchCheck for each model in a client reads it. Short-lived clients don't benefit from the cache. For small batches with few aliases, the option may save little or nothing.
- **Harder to debug:** the SDK changes what it sends. For a merged group, a server error names the final relation (`viewer`), not the one you asked for (`can_share`).
- **Risk of drift:** a ported version could read the model differently from `language` or the server. This is reduced by keeping the rules in `language`, running the shared tests, and later opening a PR in each SDK automatically when the `language` functions change.

# Alternatives
[alternatives]: #alternatives

- **Merge on the server:** this saves server work but doesn't reduce counted checks, because limits apply to the request before the server reads the model. It's still worth doing separately.
- **Each SDK writes its own rules:** they would drift apart and could disagree with the server. Keeping the rules in `language` with shared tests avoids that.
- **A standalone code snippet for developers:** no one maintains it, and every app re-implements subtle rules.
- **Do nothing:** apps keep choosing between fine-grained permissions and fewer counted checks.

# Prior Art
[prior-art]: #prior-art

- The server already merges exact repeats within one BatchCheck (OpenFGA v1.8.1, [openfga/openfga#2102](https://github.com/openfga/openfga/pull/2102)). It doesn't merge aliases, and it runs after the request has been counted.
- The [Queries with Usersets](https://github.com/openfga/rfcs/blob/main/20240328-queries-with-usersets.md) RFC covers how the server answers checks for userset users.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

**To settle before merging:**

1. **Names.** The option merges exact repeats too, so `optimize_relation_aliases` (from the issue) may not fit. Something like `merge_equivalent_checks`? Also the response field (`checks_sent`?). Each SDK would use its own casing.
2. **Function names and package** in `language`.
3. **Beta release:** a pre-release version (e.g. `0.11.0b1`), or a normal release with the option marked beta?

**Out of scope, possible follow-ups:**

- Using the model cache in other SDK methods.
- ClientBatchCheck and ListRelations.
- Other SDKs, and JS and Java versions of the functions in `language`.
- Opening a PR in each SDK automatically when the `language` functions change.
- Merging overlapping relations, like `can_view: can_edit or viewer`.
