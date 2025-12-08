# Meta
[meta]: #meta
- Name: gRPC Datastore
- Start Date: 2025-12-08
- Author: @mihai-turdean
- Status: Draft
- RFC Pull Request:
- Relevant Issues: n/a
- Supersedes: n/a

## Summary
[summary]: #summary

This RFC proposes adding a gRPC-based storage backend to OpenFGA. This introduces a defined gRPC interface (`StorageService`) that mirrors the internal `OpenFGADatastore` interface, allowing the storage layer to be decoupled from the main OpenFGA binary. It includes a client implementation within OpenFGA to connect to remote storage servers, and a reference standalone server implementation.

## Definitions
[definitions]: #definitions

*   **gRPC**: [gRPC Remote Procedure Call](https://grpc.io/) is a high-performance, open-source universal RPC framework. It uses Protocol Buffers as its interface description language.

## Motivation
[motivation]: #motivation

Currently, OpenFGA supports several built-in storage engines (Postgres, MySQL, SQLite, Memory) which are compiled directly into the OpenFGA binary. While this works well for many standard use cases, it presents challenges for:

1.  **Custom/Proprietary Backends**: Users with specialized storage needs (e.g., internal databases, proprietary distributed systems) cannot easily plug them in without forking OpenFGA.
2.  **Language Independence**: Storage adapters must be written in Go. A gRPC interface allows implementing storage backends in any language supported by gRPC.
3.  **Architecture Flexibility**: Decoupling storage allows for sidecar patterns, storage proxies, or connecting to storage services over the network, enabling new deployment models.
4.  **Separation of Concerns**: It cleanly separates the core OpenFGA logic from storage implementation details, potentially simplifying the core codebase and dependencies.

## What it is
[what-it-is]: #what-it-is

This feature provides a way to run the storage layer of OpenFGA as a separate process or service, communicating via gRPC. 

It introduces:
*   **A defined Protocol**: The `StorageService` definition in Protobuf.
*   **A new Datastore Engine**: A built-in `grpc` datastore in OpenFGA that acts as a client.
*   **A Reference Server**: A standalone `grpc-storage-server` that can wrap existing drivers (like Postgres) to expose them over gRPC.

Example Configuration:
Instead of configuring a database URI directly, a user starts OpenFGA with:
`--datastore-engine=grpc --datastore-grpc-addr=localhost:50051`

And runs a separate storage process listening on port 50051.

## How it Works
[how-it-works]: #how-it-works

The design consists of three main components:

1.  **gRPC Protocol Definition**: A formal Protobuf definition of the storage interface.
2.  **gRPC Storage Client**: A new datastore engine (`grpc`) in OpenFGA that acts as a client to the remote service.
3.  **gRPC Storage Server**: A standalone server reference implementation that exposes existing storage backends over gRPC.

### Protocol Definition

The core of this proposal is the `storage.v1.StorageService` gRPC service. It maps 1:1 with the `storage.OpenFGADatastore` Go interface.

Key RPCs include:
- `Read`, `ReadPage`, `ReadUserTuple`, `ReadUsersetTuples`, `ReadStartingWithUser` for reading tuples.
- `Write` for atomic writes/deletes of tuples.
- `ReadAuthorizationModel`, `WriteAuthorizationModel`, etc. for model management.
- `ReadChanges` for changelogs.
- `CreateStore`, `DeleteStore`, `GetStore`, `ListStores` for store management.
- `WriteAssertions`, `ReadAssertions` for testing.

The proto definitions also reuse standard OpenFGA types (Tuple, AuthorizationModel, etc.) to ensure consistency.

### Configuration

A new datastore engine type `grpc` is introduced. Configuration is handled via flags/environment variables, consistent with existing datastores.

New configuration flags:
- `--datastore-engine=grpc`
- `--datastore-grpc-addr`: Address of the gRPC server (supports `host:port` for TCP and `unix:///path` for Unix sockets).
- `--datastore-grpc-tls-cert`: Path to TLS certificate (for secure connections).
- `--datastore-grpc-tls-key`: Path to TLS key.
- `--datastore-grpc-keepalive-time`: Configuration for gRPC keepalives.

### Deployment Models

1.  **Remote Storage**: OpenFGA connects to a remote gRPC server over TCP.
2.  **Sidecar / Local Proxy**: OpenFGA connects to a local process via Unix Domain Sockets (UDS) for low latency. This is particularly useful if the storage logic requires complex local processing or caching not native to OpenFGA.

### Proto Interface

The proto file `pkg/storage/grpc/proto/storage/v1/storage.proto` defines the service. It uses streaming for operations that return iterators in the Go interface (e.g., `Read`), ensuring memory efficiency.

```protobuf
service StorageService {
  rpc Read(ReadRequest) returns (stream ReadResponse);
  rpc Write(WriteRequest) returns (WriteResponse);
  // ... other methods mirroring OpenFGADatastore
}
```

### Error Handling

Storage errors are mapped to gRPC status codes. A `StorageErrorReason` enum is defined to provide precise error causes (e.g., `COLLISION`, `NOT_FOUND`, `TRANSACTIONAL_WRITE_FAILED`) which are returned in `google.rpc.ErrorInfo` details. This allows the client to reconstruct the exact OpenFGA error types.

### Standalone Server

A reference implementation `cmd/grpc-storage-server` is provided. It:
- Accepts a standard OpenFGA datastore URI (e.g., Postgres).
- Exposes the `StorageService` gRPC API.
- Can run as a Docker container.

## Migration
[migration]: #migration

No direct migration is necessary for existing users who continue using the built-in datastores. This change is purely additive.

## Drawbacks
[drawbacks]: #drawbacks

- **Performance Overhead**: Introducing a gRPC hop adds serialization/deserialization and network/socket overhead compared to in-process function calls. Usage of Unix Domain Sockets minimizes this for local deployments.
- **Complexity**: It adds another moving part to the system if used (another service to manage/monitor).

## Alternatives
[alternatives]: #alternatives

- **Native Drivers Only (Status Quo)** Best performance, simplest deployment but requires recompilation.

## Unresolved Questions
[unresolved-questions]: #unresolved-questions

- Duplication of the datastore interface proto definitions. Since there isn't a standard way (that I know of) to import a proto definition from an external repository, my preferred solution is to move the definition to the openfga/api repo.
