# Meta
[meta]: #meta
- Name: Support for MSSQL storage
- Start Date: 2025-10-31
- Author(s): Criteo (@jamar-criteo)
- Status: Draft
- RFC Pull Request: [criteo-forks/openfga at heads/v1.10.1](https://github.com/criteo-forks/openfga/tree/heads/v1.10.1) (PoC)
- Relevant Issues: N/A
- Supersedes: N/A

# Summary
[summary]: #summary

This RFC proposes extending OpenFGA's storage backend support to include Microsoft SQL Server (MSSQL).

This addition will enable organizations that rely on MSSQL for their infrastructure to adopt OpenFGA without needing to introduce and maintain a separate database system. (cf. [popularity ranking of database management systems](https://db-engines.com/en/ranking)).

# Definitions
[definitions]: #definitions

**MSSQL**: [Microsoft SQL Server](https://www.microsoft.com/en-us/sql-server), a relational database management system developed by Microsoft.

**ORM**: Object-Relational Mapping, a programming technique for converting data between incompatible systems using object-oriented programming languages.

**SQL Dialect**: Variations in SQL syntax and features across different database systems.

# Motivation
[motivation]: #motivation

> Why should we do this?

OpenFGA currently supports a limited set of storage backends (e.g., PostgreSQL, MySQL, SQLite). Many enterprise environments rely heavily on MSSQL due to existing infrastructure, compliance, or operational preferences.

Supporting MSSQL will:
- Expand OpenFGA’s adoption in enterprise environments.
- Reduce the barrier to entry for teams already using MSSQL.
- Align with OpenFGA’s goal of being a flexible and extensible authorization platform.

> What use cases does it support?

Enterprises with existing MSSQL infrastructure seeking to integrate OpenFGA.

> What is the expected outcome?

The outcome of this RFC would be:
- A fully functional MSSQL storage backend for OpenFGA.
- Parity with existing supported backends in terms of features.
- Documentation support for users.

# What it is
[what-it-is]: #what-it-is

This feature introduces a new storage connector for Microsoft SQL Server. It enables OpenFGA to store and retrieve authorization models, tuples, assertions and changes using MSSQL as the backend.

# How it Works
[how-it-works]: #how-it-works

## SQL Server drivers

OpenFGA uses the golang `database/sql` package ([link](https://pkg.go.dev/database/sql)), which provides a standard interface to interact with [native SQL drivers](https://go.dev/wiki/SQLDrivers), streamlining database interactions within Go applications. It is designed with modularity and extensibility in mind, allowing developers to build on top of a robust SQL foundation.

By default, OpenFGA recommends PostgreSQL as its storage layer. For new storage packages, built on top of `database/sql`, it’s natural to fork the PostgreSQL implementation as a starting point.

This approach ensures compatibility with PostgreSQL features while enabling customization and optimization for specific use cases.

To establish a connection to a MSSQL server, we just provide the driver’s name _(the rest of the code to establish a connection stays identical)_: 

```golang
db, err := sql.Open("sqlserver", uri)
if err != nil {
	return nil, fmt.Errorf("initialize mssql connection: %w", err)
}
```

## Contract compliance

The MSSQL connector will be implemented as a new storage backend in the OpenFGA codebase. It will implement the existing OpenFGA storage interface ([OpenFGADatastore contract](https://github.com/openfga/openfga/blob/main/pkg/storage/storage.go#L389)), so that it can be integrated seamlessly as an additional storage backend.

This contract implementation is guaranteed at compilation time by:

```golang
// Ensures that Datastore implements the OpenFGADatastore interface.
var _ storage.OpenFGADatastore = (*Datastore)(nil)
```

<u>Full implementation</u>: [source code](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/pkg/storage/mssql/mssql.go#L56)

## Testing

Additionally, all database related tests (executed against a data storage container) will be extended to run against a [MSSQL container](https://hub.docker.com/r/microsoft/mssql-server/). ([testfixture/storage](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/pkg/testfixtures/storage/mssql.go#L26))

| Test                                                     | Postgres equivalent (from OpenFGA codebase)                                                                                                                                           | MSSQL equivalent (from Criteo’s fork)                                                                                                                                |
|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `TestServerMetricsReporting`                             | [openfga/cmd/run/run_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/cmd/run/run_test.go#L798)                                              | [criteo-forks/cmd/run/run_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/cmd/run/run_test.go#L795)                                              |
| `TestValidationResult`                                   | [openfga/cmd/validatemodels/validate_models_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/cmd/validatemodels/validate_models_test.go#L22) | [criteo-forks/cmd/validatemodels/validate_models_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/cmd/validatemodels/validate_models_test.go#L22) |
| `TestServerNotReadyDueToDatastoreRevision`               | [openfga/pkg/server/server_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/pkg/server/server_test.go#L286)                                  | [criteo-forks/pkg/server/server_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/pkg/server/server_test.go#L287)                                  |
| `TestServerWith{Storage}Datastore`                       | [openfga/pkg/server/server_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/pkg/server/server_test.go#L387)                                  | [criteo-forks/pkg/server/server_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/pkg/server/server_test.go#L426)                                  |
| `TestServerWith{Storage}DatastoreAndExplicitCredentials` | [openfga/pkg/server/server_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/pkg/server/server_test.go#L396)                                  | [criteo-forks/pkg/server/server_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/pkg/server/server_test.go#L444)                                  |
| `BenchmarkOpenFGAServer `                                | [openfga/pkg/server/server_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/pkg/server/server_test.go#L804)                                  | [criteo-forks/pkg/server/server_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/pkg/server/server_test.go#L853)                                  |
| `TestMigrateCommandRollbacks`                            | [openfga/pkg/storage/migrate/migrate_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/pkg/storage/migrate/migrate_test.go#L19)               | [criteo-forks/pkg/storage/migrate/migrate_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/pkg/storage/migrate/migrate_test.go#L20)               |
| `TestMatrix{Storage}`                                    | [openfga/tests/check/check_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/tests/check/check_test.go#L37)                                   | [criteo-forks/tests/check/check_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/tests/check/check_test.go#L41)                                   |
| `TestCheck{Storage} `                                    | [openfga/tests/check/check_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/tests/check/check_test.go#L66)                                   | [criteo-forks/tests/check/check_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/tests/check/check_test.go#L74)                                   |
| `TestMatrix{Storage}`                                    | [openfga/tests/listobjects/listobjects_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/tests/listobjects/listobjects_test.go#L20)           | [criteo-forks/tests/listobjects/listobjects_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/tests/listobjects/listobjects_test.go#L24)           |
| `TestListObjects{Storage}`                               | [openfga/tests/listobjects/listobjects_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/tests/listobjects/listobjects_test.go#L50)           | [criteo-forks/tests/listobjects/listobjects_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/tests/listobjects/listobjects_test.go#L58)           |
| `TestListUsers{Storage}`                                 | [openfga/tests/listusers/listusers_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/tests/listusers/listusers_test.go#L35)                   | [criteo-forks/tests/listusers/listusers_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/tests/listusers/listusers_test.go#L39)                   |
| `Individual tests for the data storage connector`        | [openfga/pkg/storage/postgres/postgres_test.go](https://github.com/openfga/openfga/blob/1e79eba0a65392fe0cc04dbd20f1ff2e193c3830/pkg/storage/postgres/postgres_test.go)               | [criteo-forks/pkg/storage/mssql/mssql_test.go](https://github.com/criteo-forks/openfga/blob/heads/v1.10.1/pkg/storage/mssql/mssql_test.go)                           |

## Migrations

Migrations are applied through [Goose](https://github.com/pressly/goose), the MSSQL storage solution will rely on the same migration package. _(no changes to the migration package are expected)_

To provide migrations, we will use equivalent SQL types when setting up the SQL tables.

| Postgres type   | MySQL                    | MSSQL            |
|-----------------|--------------------------|------------------|
| `TEXT`          | `VARCHAR(X)` / `CHAR(X)` | `NVARCHAR(X)`    |
| `TIMESTAMPTZ`   | `TIMESTAMP`              | `DATETIME2` *    |
| `BYTEA`         | `BLOB` / `LONGBLOB`      | `VARBINARY(MAX)` |

* Since UTC based datetime are stored, `DATETIME2` type is enough. The time zone information does not need to be stored, which would require more storage space.

The migration equivalent for MSSQL is accessible here: [criteo-forks/assets/migrations/mssql](https://github.com/criteo-forks/openfga/tree/heads/v1.10.1/assets/migrations/mssql)

## Statement building

To create dynamic SQL queries, OpenFGA uses an ORM framework named [Squirrel](https://github.com/Masterminds/squirrel).

Squirrel is a great solution when it comes to generate “basic” SQL queries, but when it comes to some SQL engine specifics, developers have to be “creative” to build some queries.

Most of the OpenFGA SQL queries would be retro-compatible with the MSSQL SQL syntax, but some need to be specific to the new connector since MSSQL does not support:
- `NOW()` (replaced by `SYSUTCDATETIME()`)
- `nil` being mapped to a `VARBINARY(MAX)` column through Squirrel causes issues since it is considered as a `NVARCHAR` (`sq.Expr("NULL")` will be used)
- `.Limit()` is not supported on MSSQL (replaced by `.Suffix("OFFSET 0 ROWS FETCH FIRST %d ROWS ONLY")`)
- Tuple matching (`WHERE (col1, col2) IN ((val1, val2), (val3, val4))`) will be replaced by `AND` / `OR` combinations.
- `INSERT INTO … RETURNING column` is not supported, will be replaced by a transaction call or `INSERT INTO … OUTPUT INSERTED.column`
- `INSERT INTO … ON CONFLICT (col) DO UPDATE SET` should be updated with a `MERGE` statement or a `DELETE` + `INSERT` statement.

## Limitations
- Since some SQL Server version [are going end-of-life](https://endoflife.date/mssqlserver), the minimal version to support is MSSQL 2022.
- Today, the double connection (read vs write)'s test had not been tested due to the fact it’s harder to setup a local replication during test.
- By design, each DB engine has limitation on clustered index size, MSSQL is limited to `900 bytes` (which is not compatible with the Primary key of the `tuple` table; composed of 5 columns). A non-clustered index will be used instead.
  - For details: [Maximum Capacity Specifications for SQL Server - SQL Server | Microsoft Learn](https://learn.microsoft.com/en-us/sql/sql-server/maximum-capacity-specifications-for-sql-server?view=sql-server-ver17) (search for `Bytes per index key`)

# Migration
[migration]: #migration

## Compatibility Breaks
- No migration for existing users since this RFC is an additive change.

## Migration Steps
- For users migrating from another backend to MSSQL:
  - Stop OpenFGA write operations  
    Export existing data using DB export tools
  - Import data into MSSQL using provided scripts or tools.
  - Update configuration to point to the MSSQL backend.
  - Restart OpenFGA write operations

The transition would cause a downtime to the system (since there is no double-write; to two different backends; feature in OpenFGA), but we don’t foresee any users going through this process.

# Drawbacks
[drawbacks]: #drawbacks

- Increased maintenance burden for supporting another backend _(which would be mitigated by Criteo’s contribution to openFGA’s project)_. 
- Potential performance tuning challenges due to MSSQL’s differences from PostgreSQL/MySQL.

# Alternatives
[alternatives]: #alternatives

- Implement the MSSQL connector as a dedicated storage plugin.
  - Modularization of storage package should be a dedicated RFC on its own. It is decorrelated from the MSSQL connector implementation. The important consideration here, is that all storage packages follow the same contract.

# Prior Art
[prior-art]: #prior-art

N/A

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

It would be nice, to decouple storage implementation through modularization, but since the modularized storage implementation is not available, we are requesting to upstream the connector’s source code. If modularization is a pre-requisite, can we have an estimate when it would be achieved?
