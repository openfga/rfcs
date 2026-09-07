# Meta
[meta]: #meta
- Name: mysql-secondary-db-support
- Start Date: 2026-09-07
- Author(s): chiragsoni-eternal
- Status: Draft
- RFC Pull Request: (leave blank)
- Relevant Issues:
  - https://github.com/openfga/openfga/issues/3282
- Supersedes: N/A

# Summary
[summary]: #summary

Add secondary-DB (read replica) support to the MySQL datastore, matching what already exists for PostgreSQL: writes go to the primary DB, and reads that don't require high consistency go to the secondary DB, using the same consistency-based routing logic as the PostgreSQL datastore.

# Definitions
[definitions]: #definitions

- `secondary db`: Its a read replica of DB where we can do read calls

# Motivation
[motivation]: #motivation

- Why should we do this?
The MySQL datastore currently supports only a single primary DB, which prevents scaling read traffic independently of writes. This matters because permissions change infrequently, but Check/authorization traffic dominates load on any OpenFGA server.

- What use cases does it support?
Routing read operations to separate MySQL read-replica nodes.

- What is the expected outcome?
Users running MySQL as their OpenFGA datastore can handle read-heavy load by horizontally scaling MySQL read replicas.

# What it is
[what-it-is]: #what-it-is

Target persona: platform operators running MySQL as their OpenFGA datastore who want to scale read (Check/List) traffic independently of writes.

Secondary-DB config already exists for PostgreSQL, so users familiar with that datastore already know this pattern; new users get the same widely-adopted read-replica pattern.

# How it Works
[how-it-works]: #how-it-works

Mirrors the existing PostgreSQL secondary-DB support:
- Writes always go to the primary DB.
- Reads with `HIGHER_CONSISTENCY` go to the primary DB.
- Reads with `MINIMIZE_LATENCY` (the default) go to the secondary DB, when one is configured.
- Metadata reads (`GetStore`, `ListStores`, `ReadAssertions`, `ReadChanges`, `ReadAuthorizationModel(s)`, `FindLatestAuthorizationModel`) always request `MINIMIZE_LATENCY`, same as PostgreSQL today.
- No failover: if the secondary is configured but unreachable, affected reads fail rather than falling back to primary.

Configuration reuses the existing shared flags/env vars: `--datastore-secondary-uri` / `-username` / `-password` (`OPENFGA_DATASTORE_SECONDARY_URI` / `_USERNAME` / `_PASSWORD`), the same ones PostgreSQL already uses. Unset by default, so existing deployments are unaffected.

# Migration
[migration]: #migration

Non-breaking, opt-in. Existing deployments without `datastore-secondary-uri` set are unaffected. To adopt, set `--datastore-secondary-uri` (and credentials, if different from primary) to point at a MySQL read replica. No schema or data migration needed.

# Drawbacks
[drawbacks]: #drawbacks

Adds a second routing/pool-setup/readiness-check code path to the MySQL datastore that maintainers need to keep correct going forward (mirrors the existing PostgreSQL implementation, so the pattern is already proven).

# Alternatives
[alternatives]: #alternatives

- What other designs have been considered?
  - Vertically scaling the primary writer — doesn't address read scaling and hits a hardware ceiling.
  - A sharded MySQL cluster with multiple writers — unnecessary complexity given write volume is low relative to reads.

- Why is this proposal the best?
Matches the existing, proven PostgreSQL pattern rather than introducing a new mechanism, keeping the two datastores consistent.

- What is the impact of not doing this?
Users are stuck vertically scaling the primary or sharding, neither of which fits a read-heavy authorization workload.

# Prior Art
[prior-art]: #prior-art

Mirrors the existing PostgreSQL secondary-DB/read-replica support in `storage/postgres`.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

N/A — this RFC mirrors existing, already-accepted PostgreSQL secondary-DB behavior. Any change to that behavior would apply to both datastores and is out of scope here.