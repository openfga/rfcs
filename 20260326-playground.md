# OpenFGA Playground

## Meta

- **Name**: OpenFGA Playground
- **Start Date**: 2026-03-26
- **Author(s)**: [rhamzeh](https://github.com/rhamzeh)
- **Status**: Draft
- **RFC Pull Request**: [35](https://github.com/openfga/rfcs/pulls/35)
- **Relevant Issues**:
  - [openfga/roadmap#7](https://github.com/openfga/roadmap/issues/7): Open Source play.fga.dev
  - [openfga/openfga#338](https://github.com/openfga/openfga/issues/338): CORS errors running playground locally
  - [openfga/openfga#2488](https://github.com/openfga/openfga/issues/2488): Playground only works on Chrome
  - [openfga/openfga#2503](https://github.com/openfga/openfga/issues/2503): Request for externalized assets folder
  - [openfga/openfga#1882](https://github.com/openfga/openfga/issues/1882): Playground not running on Brave
  - [Playground-labeled discussions](https://github.com/orgs/openfga/discussions?discussions_q=is%3Aopen+label%3APlayground)
- **Supersedes**: N/A

## Summary

The OpenFGA Playground is a browser-based environment for authoring authorization models, managing relationship tuples, running assertions, visualizing model graphs and interacting with an FGA Store. It will be a new, open-source, locally-run playground that replaces the Playground built into the OpenFGA image, which is just an iframe around the hosted playground at `play.fga.dev`.

## Definitions

- **Authorization model**: A schema written in the OpenFGA DSL that defines the types and relations in an authorization system.
- **Tuple / relationship tuple**: A triple `(user, relation, object)` representing a concrete relationship between entities.
- **Assertion**: A declared expectation about a Check query of whether a given `(user, relation, object)` combination should be `allowed` or `not allowed` and is used for automated testing of the model.
- **Store**: An isolated namespace in OpenFGA containing one or more model versions and all associated tuples.
- **DSL**: The OpenFGA Domain Specific Language for writing authorization models in human-readable form.
- **Web Component**: A browser-native custom element standard (`customElements.define`) that works in any HTML context without a JavaScript framework.
- **`fga serve`**: A new subcommand on the `fga` CLI that runs a local HTTP proxy, handles secret storage, and proxies API requests to upstream OpenFGA servers.
- **WASM (WebAssembly)**: A binary instruction format that allows compiled programs (e.g., Go) to run in the browser.

## Motivation

### Developers need an interactive environment to explore and validate authorization models

Building an authorization model for a non-trivial system is inherently iterative. Developers need to:

- Write a model and immediately see how types and relations connect as a graph.
- Add sample relationship tuples to represent real-world state.
- Run check assertions against the model and data to verify that the model behaves as intended before deploying it.
- Compare two model versions side-by-side to understand what changed between iterations.
- Run queries such as ListObjects, ListUsers and (the SDK-only) ListRelations in order to build, verify and understand their model.
- Look at the changelog and read tuples to understand when a tuple was added/deleted.
- See a log of API requests to understand how their actions translate to API requests.
- Import from and export to the store yaml file.
- Be able to connect to and switch between multiple stores.

While we have been pushing folks to use VSCode with the OpenFGA extension in combination with the CLI, the demand for a Playground has not subsided. Users have cited the model graph in particular as an asset when learning the OpenFGA authorization model or explaining the model to coworkers.

### The existing hosted playground is not sufficient

We have maintained a hosted playground at `play.fga.dev`. While it served as a starting point, it has several significant limitations:

1. **It is not open source.** The codebase is proprietary, tied to Auth0 FGA and cannot be contributed to by the community, audited, or self-hosted. We are asking for significant trust from the community to use a tool they have no access to audit. It also flies against what OpenFGA is: an Open Source, collaborative community project in the CNCF ecosystem.

2. **It is embedded via iframe.** To work around CORS limitations when running against a local OpenFGA server, the playground is loaded inside an iframe hosted on the same origin as the playground's own backend. This embedding technique has become increasingly fragile:
   - Recent versions of Chrome and Safari enforce stricter mixed-content policies, blocking HTTP requests (to a local OpenFGA server) from an HTTPS-hosted page. Users connecting to `http://localhost:8080` from the HTTPS-hosted `play.fga.dev` receive opaque network errors with no clear remediation path.
   - Cross-origin cookies and storage restrictions (`SameSite`, third-party cookie blocking) interfere with session management in embedded iframes.
   - The iframe boundary makes it impossible to integrate the playground cleanly into developer tools like VS Code extensions or documentation sites.

3. **CORS friction.** Developers connecting the existing playground to self-hosted or cloud-hosted OpenFGA servers routinely encounter CORS errors because OpenFGA servers must be configured to explicitly allow the playground's origin. This is a recurring source of user frustration in community channels.

4. **Limited functionality.** The existing playground lacks model version history, model diffing, list users, list objects and other helpful tools which developers need in practice to build and understand their model.

5. **Low modularity/reusability.** The existing playground UI is monolithic and is hard to split apart. Embedding just a model editor (with DSL syntax highlighting) in documentation, or just a model graph in a VS Code webview, is not possible. We have made some work in the past by extracting some amount of logic into `@openfga/frontend-utils`, but the components themselves are not reusable.

6. **An effort to maintain.** The current playground is built on Next.js + React. That ecosystem has proven hard to maintain over the years, requiring an outsized amount of work for dependency upgrades, security advisories, and ecosystem churn.

7. **Low information density.** The Playground UI has a very small window for the model, making the task of making changes harder than it should be. Similarly the tuple editor only allows seeing one or two tuples at a time on smaller screens.

8. **Graph usability.** The current graph, while helpful to many, is not entirely accurate and is very limited. We need a way to allow having different graph representations people can switch between and even contribute to, as each graph may be able to focus on a particular aspect without having to represent all the data.

### Community feedback around these pain points

The OpenFGA community has raised them repeatedly across GitHub issues and discussions:

#### Browser compatibility and CORS failures

- [openfga/openfga#338](https://github.com/openfga/openfga/issues/338): Running the playground locally iframes `play.fga.dev` instead of serving a local UI. This causes CORS errors on any hostname other than `localhost:3000`, blocks deployment on custom ports or domains, and fails under strict CSP policies.
- [openfga/openfga#2488](https://github.com/openfga/openfga/issues/2488): The playground only works in Chrome. Safari and Brave users report blank screens with no errors, likely due to stricter mixed-content and third-party iframe policies.
- [openfga/openfga#1882](https://github.com/openfga/openfga/issues/1882): Brave browser blocks playground requests (`ERR_BLOCKED_BY_CLIENT`) due to its built-in privacy shields. No workaround exists.
- [Discussion #516](https://github.com/orgs/openfga/discussions/516): CORS errors when creating a store from the local playground, even with CORS configured in Docker Compose.

#### Deployment and self-hosting gaps

- [openfga/openfga#2503](https://github.com/openfga/openfga/issues/2503): Request for a flag to serve custom static playground content instead of assets bundled in the binary, enabling lightweight Docker images. (Implicitly addressed by this RFC: the playground is a standalone web app, hosted it however one likes, pointing at the API.)
- [Discussion #270](https://github.com/orgs/openfga/discussions/270): Questions about the source code for the embedded playground; not available because the current playground is not open source.
- [Discussion #200](https://github.com/orgs/openfga/discussions/200): Issues deploying the playground in Kubernetes via Helm chart.

#### Missing features requested by the community

- [Discussion #18](https://github.com/orgs/openfga/discussions/18): Export and import of a full store (model + tuples + assertions). (Addressed by this RFC's YAML import/export.)
- [Discussion #404](https://github.com/orgs/openfga/discussions/404): Conditions support in tuples and checks within the playground.
- [Discussion #299](https://github.com/orgs/openfga/discussions/299): Visualization of how type definitions compose relationships across multiple degrees.
- [Discussion #122](https://github.com/orgs/openfga/discussions/122): Graph showing relations between types as nodes and arrows (like Neo4j).
- [Discussion #107](https://github.com/orgs/openfga/discussions/107): Graph visualization fails for AND and BUT NOT operators.

#### UX issues

- [Discussion #183](https://github.com/orgs/openfga/discussions/183): The playground clears tuples when a write request fails, losing user work.
- [Discussion #165](https://github.com/orgs/openfga/discussions/165): The "user" label in the tuple form is misleading when the subject is not a user type.

This RFC addresses all of the browser compatibility, CORS, self-hosting, and most feature request issues. Conditions support is deferred to a future iteration.

The new architecture resolves the CORS and browser compatibility issues through three complementary approaches: (1) the playground is served locally as a standalone web app, eliminating the mixed-content problems of an HTTPS-hosted page calling an HTTP local server; (2) the `ProxyBackendAdapter` (via `fga serve`) forwards all API requests server-side, removing CORS from the equation entirely; and (3) the `DirectBackendAdapter` is only used under same-origin or CORS-configured scenarios where browser restrictions do not apply.

## What We're Proposing

A new Playground that ships in several deployment surfaces and is built around a small set of design principles.

### Deployment surfaces

1. **A locally-run browser application** that developers host themselves either as a static site, via `npm run dev` during development, or from the published Docker image. It can connect directly to an API, in the future it should be able to use a WASM store or to proxy requests through the FGA CLI. Two possible future conveniences we may pursue: bundling the playground into `fga serve` itself (so a single binary serves both the API and the UI), or embedding it into the OpenFGA server binary (so `openfga server run --playground` hosts the UI). Both are out of scope for v1.

2. **A library of composable Web Components** (`@openfga/playground-components`) that can be independently imported into documentation sites, VS Code extension webviews, and third-party dashboards.

3. **A bundled optional WASM OpenFGA server** that allows the Playground to work entirely in the browser with no API dependency.

4. **A direct connection to unauthenticated OpenFGA servers** via `DirectBackendAdapter`, used when the playground is colocated with the server (same origin) or the server has CORS configured to permit it. For OpenFGA servers that require an API token or client credentials, the direct adapter is not safe (browser storage is unsuitable for secrets), for those we should consider going through the proxy described in #5.

5. **A shared connection management system through the CLI** (`fga serve`), serving as the single place developers manage their OpenFGA server connections and usable by the playground, the `fga` CLI, the VS Code extension, the IntelliJ Plugin, and any other tool in the ecosystem. On the playground side this avoids storing client credentials in the browser, and the CLI can be secured with an optional session token to prevent other local processes from accessing the FGA data through it.

### Design principles

- **Minimal dependencies.** We aim to reduce our reliance on third-party libraries, especially large frameworks like React that necessitate constant maintenance to keep up to date. The implementation uses Lit (a thin layer over native Web Components), nanostores for state, and only the minimum tooling needed to assemble them.
- **Pluggable backend.** The shell talks to a `BackendAdapter` interface, so the same UI works across all three deployment modes (direct, proxy, WASM) without code changes and third parties can plug in their own implementations.
- **Stateless components.** All UI components are stateless views that receive data via properties and emit events. The shell is the only layer that connects components to state and the backend, which keeps the components reusable in any host context.
- **Theme-able by tokens.** All components consume a CSS custom property contract (`--openfga-*`), so a host application can re-theme the playground by overriding tokens at the document or shadow root level.

### Target personas

- **Authorization model author**: A developer writing an OpenFGA model for the first time or iterating on an existing one. Primary playground user.
- **Application developer**: A developer integrating OpenFGA into an application who wants to validate that specific relationship checks behave as expected.
- **Documentation reader**: A developer reading OpenFGA documentation who encounters embedded, interactive model editors or graphs within doc pages.
- **Platform operator**: A developer managing connections to multiple OpenFGA servers (local, staging, production) through a shared configuration.

### Core features

#### Model editor with DSL/JSON toggle

The model editor provides Monaco-based syntax highlighting, autocompletion, and inline validation for the OpenFGA DSL. Developers can toggle between DSL and JSON representations. Saving (Ctrl+S) writes a new authorization model version to the server. This is equivalent to our current model editor.

#### Model graph visualization

Every time the model is updated, a graph renders the types and relations as a node-link diagram. Type nodes and relation nodes are visually distinct. Edges represent direct assignments (solid) and computed relations (dashed). The layout should be deterministic: the same model always produces the same layout. We should allow for additional alternate graphs, whether created by us or contributed by the community. The graphs should maintain similar UX, using the same tool for drawing the graph, but compute the edges and nodes differently. They should be theme-able in the same way the playground is theme-able, so that changing the relation color or moving between dark and light mode causes a consistent change throughout.

#### Tuple manager

A table showing all relationship tuples in the active store. Developers can add and remove tuples. The add form provides model-aware autocomplete for the relation field (only relations defined in the current model are suggested) and type-aware autocomplete for user, relation and object fields. This should load only a few tuples (100 by default) but allow the user to load more as they wish.

#### Assertion runner

Developers define Check assertions (user, relation, object, expected result). Each assertion can be run individually or all at once. Results are shown inline as pass (allowed matches expectation), fail (allowed does not match expectation), or error (the API returned an error). A resolution path view (showing the expand tree from the Check) should also be available. While long-term we should support the full tests supported by the CLI and the OpenFGA store file, initially we're limited by the server assertion support and that is what we are targeting.

#### Query runner

A scratchpad for ad-hoc API queries that don't fit the assertion model:

- **Check**: single Check query with user/relation/object inputs and an Expand button to view the resolution path inline
- **ListObjects**: given a user, relation, and object type, list all objects the user has the relation with
- **ListUsers**: given an object and relation, list all users with that relation
- **ListRelations**: given a user and object, list all relations the user has on that object (implemented client-side via parallel Check calls since no server endpoint exists)

These complement the assertion runner which is for declarative tests; the query runner is for exploration and debugging.

#### Resolution path / Expand viewer

A modal that visualizes the result of an Expand API call as a directed graph with a green path for the route that produced an `allowed` decision, and red for branches that didn't contribute. Triggered from any assertion result or Check query result. Helps developers understand *why* a check was allowed or denied, which is critical for debugging non-trivial models.

#### Changelog viewer

A tab that calls `ReadChanges` and displays the full history of tuple writes and deletes for the active store, with continuation token pagination. Filterable by type and start time. Helps developers understand the audit history of their store and debug "where did this tuple come from?" questions.

#### API log / dev console

A built-in network inspector that captures every OpenFGA API call made by the playground (via an axios interceptor in the adapter layer). Each entry shows method, path (with the proxy prefix stripped to show the real OpenFGA endpoint), request body, response, status code, round-trip duration, and selected headers. Expandable rows for full body inspection. Helps developers understand how their UI actions translate into API requests and is useful both for learning and for debugging.

#### Model version history

Every time a model is saved, the server creates a new authorization model version. The playground displays the full version history for the active store in a dropdown, with each version identified by its creation time (decoded from the ULID). Developers can select any prior version to load it into the editor.

#### Model diff

Developers can select two model versions to compare. The diff is displayed using Monaco's side-by-side diff editor, with DSL syntax highlighting on both sides.

#### Sample store browser

A dropdown lists all sample stores from the `openfga/sample-stores` GitHub repository. Selecting a sample fetches its `.fga.yaml` file and loads the model, tuples, and assertions into the playground. Samples are fetched at runtime so the playground always reflects the latest samples without a rebuild. These should be view only, but are importable into a new store/exportable into yaml. They cannot do any queries on them before importing, though that may be possible in the future once we have the WasmBackend.

#### YAML import/export

The current playground state (model, tuples, assertions) can be exported to a `.fga.yaml` file compatible with `fga model test`. Existing `.fga.yaml` files can be imported by file picker or drag-and-drop.

#### Server connection manager

For the proxy service, if we build it, it would most likely be part of the FGA CLI. We would introduce a new configuration in `~/.config/fga/servers.yaml` that would allow storing named server connections (URL, auth type, credentials, store ID) for multiple stores and servers. The CLI can then expose the proxy through an `fga serve` command or similar. The Playground can authenticate to it using a session token, and then be able to talk to any of the stores. The Playground should never be able to access or retrieve client secrets or an authorization tokens, though it can set them.

## How It Works

### Architecture overview

The playground shell talks to the OpenFGA API through a pluggable `BackendAdapter` interface. Three adapters ship with the project, each suited to a different deployment mode. In all three cases, the shell is identical and only the adapter changes.

```
┌───────────────────────────────────────────────────────────────────────────┐
│  Browser                                                                  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  @openfga/playground (Lit shell app)                                │  │
│  │                                                                     │  │
│  │          Components  ←→  Core  ←→  BackendAdapter                   │  │
│  │                                          │                          │  │
│  │          ┌───────────────────────────────┼───────────────────────┐  │  │
│  │          ▼                               ▼                       ▼  │  │
│  │  ┌───────────────┐           ┌───────────────────┐    ┌────────────┐│  │
│  │  │ DirectBackend │           │ WASMBackend       │    │ ProxyBack- ││  │
│  │  │ Adapter       │           │ Adapter (future)  │    │ endAdapter ││  │
│  │  │               │           │                   │    │ (future)   ││  │
│  │  │ Calls SDK     │           │ Calls WASM build  │    │            ││  │
│  │  │ directly      │           │ of OpenFGA        │    │ fetch() to ││  │
│  │  │ (same-origin  │           │ running in a      │    │ fga serve  ││  │
│  │  │ or CORS-      │           │ Web Worker with   │    │ HTTP proxy ││  │
│  │  │ configured)   │           │ no network calls  │    │            ││  │
│  │  └───────┬───────┘           └───────────────────┘    └────┬───────┘│  │
│  └──────────┼─────────────────────────────────────────────────┼────────┘  │
└─────────────┼─────────────────────────────────────────────────┼───────────┘
              │                                                 │
              │                                                 ▼
              │                                    ┌──────────────────────────┐
              │                                    │  fga serve               │
              │                                    │  (local HTTP proxy)      │
              │                                    │                          │
              │                                    │  • Server management     │
              │                                    │  • Secret storage        │
              │                                    │  • API proxying          │
              │                                    │  • Auth token exchange   │
              │                                    └───────────┬──────────────┘
              │                                                │
              ▼                                                ▼
       ┌──────────────┐                        ┌──────────────────────────────┐
       │  Upstream    │                        │  Upstream OpenFGA servers    │
       │  OpenFGA     │                        │  (local, Auth0 FGA, others)  │
       │  server      │                        │                              │
       └──────────────┘                        └──────────────────────────────┘
```

**`DirectBackendAdapter`**: Calls the OpenFGA API directly from the browser using `@openfga/sdk`. Used when the playground is embedded in the same origin as the OpenFGA server (e.g., served from the same port as a local OpenFGA instance), or when the server has CORS configured to allow the playground origin. For security reasons, this adapter intentionally does not support API token or client credentials auth as storing those in the browser is unsafe, so authenticated servers must go through the proxy adapter instead.

**`WASMBackendAdapter`**: Calls into a WebAssembly build of OpenFGA running in a Web Worker. The "server" lives entirely in the browser with no network calls. This adapter enables a fully hosted, shareable playground (e.g., `play.openfga.dev`) and an offline-capable experience. Deferred to a future iteration; gated on a WASM build of OpenFGA being available.

**`ProxyBackendAdapter`**: Calls `fga serve`, a local HTTP proxy that manages server profiles, stores secrets in `~/.config/fga/servers.yaml`, and forwards requests to upstream OpenFGA servers (local, Auth0 FGA, or any other). This enables local development with authenticated servers: it lets the user manage multiple server connections with their credentials in one place and avoids CORS issues by forwarding all requests server-side. It also avoids the issue with the DDirectBackend adapter of storing credentials in the browser. When `fga serve` has session token auth enabled (the default), the playground prompts the user for the token on first connection.

When the playground starts, the user picks a backend (or one is preconfigured by the host environment):

- **Local development against an unauthenticated OpenFGA server**: `DirectBackendAdapter` pointing at the local server URL. This is useful as a companion to a docker-compose setup, allowing folks to replace the built-in playground.
- **Local development with authenticated servers** (future): `ProxyBackendAdapter` + `fga serve` (session-token authenticated).
- **Hosted `play.openfga.dev`** (future): `WASMBackendAdapter`.

The playground is a standalone web application hosted independently of `fga serve` (as a static site, a Docker image, or a dev server) and pointed at the local proxy for API calls. Bundling the playground into `fga serve` or into the OpenFGA server binary is a possible future convenience, but not part of v1.

### Four npm packages

We are suggesting an architecture of 4 packages:

- **`@openfga/playground-core`**: Pure TypeScript. State management (nanostores), the `BackendAdapter` interface and its three implementations (`DirectBackendAdapter`, `ProxyBackendAdapter`, `WASMBackendAdapter`), YAML import/export, validation orchestration. No DOM dependencies.
- **`@openfga/playground-components`**: Lit Web Components. Stateless views that accept data via properties and emit Custom Events. Each component has a subpath export for independent import. No dependency on the core package.
- **`@openfga/frontend-utils`**: Subsumes and replaces the current `@openfga/frontend-utils`, will have the exportable, reusable logical TypeScript components and theme elements that complement the `@openfga/playground-components`.
- **`@openfga/playground`**: The Lit shell application. Composes core and components; handles event wiring, layout, toolbar widgets, and URL state.

### Data flow

Components are stateless views. All state lives in the core package (nanostores atoms). The shell is the only layer that connects components to core:

```
User action → Component emits CustomEvent
  → Shell handler calls event-handlers.ts
  → Core action updates nanostore atom
  → Adapter calls backend via @openfga/sdk
  → Shell subscribes to atom, passes updated value to component
  → Component re-renders
```

This one-way flow means components never import from core and are independently usable outside the playground.

### Web Component embedding

Each component in `@openfga/playground-components` has a subpath export:

```json
{
  "exports": {
    "./model-editor": "./dist/model-editor/index.js",
    "./model-graph": "./dist/model-graph/index.js",
    "./model-diff": "./dist/model-diff/index.js"
  }
}
```

A documentation site that needs only the model editor imports `@openfga/playground-components/model-editor` without pulling in Cytoscape (from the graph component) or any backend dependencies. Each subpath's build is isolated.

```html
<!-- In a Docusaurus MDX page -->
<script type="module">
  import '@openfga/playground-components/model-editor';
</script>
<openfga-model-editor model="model\n  schema 1.1\ntype user" readonly></openfga-model-editor>
```

### `BackendAdapter` interface

The architecture overview above describes the three adapters at a high level. This section specifies the TypeScript contract.

The core package defines a minimal `BackendAdapter` interface that every adapter must implement, plus an optional `ConnectionManager` interface for adapters that support managing multiple servers and their credentials. Splitting the contract this way means TypeScript prevents accidentally calling `createServer` on a `DirectBackendAdapter`, and third-party adapter authors get a small required surface area.

```typescript
/** Required: every adapter produces OpenFGA clients and enumerates its connections. */
interface BackendAdapter {
  /** Return the configured server connections. May be a synthetic single-entry
   *  list for adapters that don't support multi-server (Direct, WASM). */
  listServers(): Promise<ServerConfig[]>;

  /** Return an SDK client configured to talk to the given server. */
  getClient(serverId: string): OpenFgaClient;
}

/** Optional: implemented by adapters that support managing connections at runtime. */
interface ConnectionManager extends BackendAdapter {
  createServer(server: NewServer): Promise<ServerConfig>;
  updateServer(id: string, update: ServerUpdate): Promise<ServerConfig>;
  deleteServer(id: string): Promise<void>;
  addStore(serverId: string, store: NewStoreEntry): Promise<StoreEntry>;
  updateStore(serverId: string, storeId: string, update: StoreEntryUpdate): Promise<void>;
  removeStore(serverId: string, storeId: string): Promise<void>;
}
```

| Adapter | `BackendAdapter` | `ConnectionManager` |
| --- | --- | --- |
| `DirectBackendAdapter` | yes | no  (single-server, configured at startup) |
| `ProxyBackendAdapter` | yes | yes |
| `WASMBackendAdapter` | yes | no (single in-browser instance) |

The shell uses capability detection to gate the connection management UI:

```typescript
function isConnectionManager(a: BackendAdapter): a is ConnectionManager {
  return 'createServer' in a;
}

if (isConnectionManager(this._adapter)) {
  // show "Add server" / "Edit server" / "Delete server" UI
}
```

For non-`ConnectionManager` adapters, `listServers()` returns a single synthetic entry (e.g., `{ id: 'direct', name: 'OpenFGA Server', api_url: '...' }` or `{ id: 'wasm', name: 'In-Browser', api_url: 'wasm://' }`) so the rest of the shell (store selector, model editor, tuple manager, etc..) works uniformly across all three deployment modes.

A third party can implement `BackendAdapter` (and optionally `ConnectionManager`) backed by their own internal API and reuse `@openfga/playground-components` and `@openfga/playground-core` without `fga serve` if they so wish.

### `fga serve` proxy

The local proxy runs on `http://localhost:8880` (configurable via `--host` and `--port`) and exposes:

| Endpoint | Description |
| --- | --- |
| `GET /servers` | List configured servers (secrets redacted) |
| `POST /servers` | Add a server connection (accepts secrets, stores on disk in `~/.config/fga/servers.yaml`) |
| `PUT /servers/:id` | Update a server connection |
| `DELETE /servers/:id` | Remove a server connection |
| `ANY /servers/:id/proxy/*` | Transparent proxy to the upstream OpenFGA server |
| `GET /healthz` | Health check (always accessible) |

The browser configures `@openfga/sdk`'s `OpenFgaClient` with `apiUrl: "http://localhost:8880/servers/:id/proxy"`. All standard OpenFGA API calls (Check, Write, Read, WriteAuthorizationModel, etc.) flow through the proxy unchanged. The proxy attaches the appropriate auth headers: none, a static API token, or a bearer token obtained via OAuth 2.0 client credentials exchange (with caching). The browser itself never holds secrets.

**Playground hosting**. The proposed `fga serve` is API-only for now and does not serve the playground UI. The playground is a separate web application that the user runs independently and points at the local `fga serve` instance for its API calls. This keeps the CLI binary small and decouples playground release cadence from the CLI.

We may choose to add an optional `--playground` flag (or similar) in a future iteration that serves a bundled playground build directly from `fga serve`, for a single-binary, zero-setup experience.

Neither the proxy nor the embedded playground is part of v1, but the proxy can be a fast follow.

**Security layers**:

1. **Localhost binding**. The proxy binds to `localhost` by default; remote network access is impossible without an explicit `--host` flag.
2. **Origin validation**. Browser requests from any `Origin` other than `localhost`, `127.0.0.1`, or `::1` are rejected, preventing malicious pages from making cross-origin requests to the local server.
3. **Session token authentication**. By default, `fga serve` generates a random session token at startup and prints it. All requests (except `/healthz`) must include the token via the `X-Serve-Token` header or `?token=` query parameter. The playground reads the token from the URL on first load, persists it in `sessionStorage`, and includes it in every subsequent request. A user-supplied token can be set with `--token <value>`, or token auth can be disabled entirely with `--no-token` (not recommended). This protects against other local processes (including malicious browser tabs not protected by the Origin check) using the configured server credentials.
4. **Request body limits and timeouts**. Request bodies are capped at 10 MB; upstream HTTP calls have 30s timeouts. APIURL values are validated to require an `http`/`https` scheme and a non-empty host before being stored.

**Note**: The `fga serve` proxy itself is still in ideation and would not be part of the initial playground release.

### Shared connection config

`fga serve` reads and writes `~/.config/fga/servers.yaml`. The same file can be used by `fga check`, `fga tuple write`, the VS Code extension, and any other OpenFGA tool that adopts the format. A developer who adds a server via `fga server add` immediately sees it in the playground (and vice versa) without any additional setup. Stores may override or extend some of the server configuration.

```yaml
version: 1
servers:
  - id: local-dev
    name: Local OpenFGA
    api_url: http://localhost:8080
    auth:
      method: none
    capabilities:
      store_crud: true
      list_models: true
    stores:
      - store_id: 01KNKVN03BMAJVN3ERX7S643CD
        alias: store-1
  - id: api-token-secured
    name: API Token Secured OpenFGA
    api_url: https://api.fga.example
    auth:
      method: api_token
      api_token: "${OPENFGA_API_TOKEN}"   # stored in config after first setup via fga serve
    capabilities:
      store_crud: true
      list_models: true
    stores:
      - store_id: 01KNKVMKYTJFEANM416DSATMAM
        alias: store-2
  - id: auth0-fga
    name: Auth0 FGA
    api_url: https://api.us1.fga.dev
    auth:
      method: client_credentials
      api_token_issuer: "auth.fga.dev"
      api_audience: "https://api.us1.fga.dev/"
    capabilities:
      store_crud: false
      list_models: true
    stores:
      - store_id: 01KNKVMDXZNZ9WN3EYGBMFWWEG
        alias: store-3
        auth:
           # method is inherited from the server config, no need to respecify
           client_id: clientid3
           client_secret: clientsecret3
      - store_id: 01KNKVM6QZ71AK61F7VHTQ1SX8
        alias: store-4
        auth:
           # method is inherited from the server config, no need to respecify
           client_id: clientid4
           client_secret: clientsecret4
```

The `capabilities` block drives UI gating: the "New Store" button is hidden and the store listing call is skipped when `store_crud` is false.

## Drawbacks

- **Local daemon for the recommended path.** The recommended workflow for connecting to authenticated OpenFGA servers requires installing the `fga` CLI and running `fga serve`. This is a new step that existing CLI users will need to learn. Unauthenticated local servers can use `DirectBackendAdapter` without `fga serve`, and the future WASM adapter removes the dependency entirely, but for the most common case (developer iterating against a local or remote OpenFGA server with credentials), the daemon is required. The playground shows a clear error message with setup instructions when the proxy is not reachable.

- **No hosted / shareable playground until WASM ships.** Until the WASM adapter is available, there is no way to share a URL that captures playground state across devices. Sharing is done by exporting a `.fga.yaml` file in the meantime.

- **Bundle size.** The built playground bundles Monaco (~2 MB gzipped) and Cytoscape. The WASM adapter, when added, will further increase the initial download. Lazy-loading and code splitting mitigate the user-facing impact but can't eliminate it.

- **CLI binary inflation if we embed in the future.** If we later choose to bundle the playground into `fga serve` (or `openfga server run`) for a single-binary experience, the binary will grow by the playground bundle size. We would want to either offer a "lite" build without the embedded playground for users who only need the CLI commands, or keep the embedded build behind an opt-in flag.

- **No server-side rendering.** Web Components require JavaScript to instantiate, so any host that needs server-side rendering (notably our documentation site) cannot pre-render component contents. The components are progressively enhanced, as in the surrounding HTML still renders, but interactive content like the model graph appears only after JS loads.

- **Secrets stored as plaintext on disk.** `fga serve` stores credentials (API tokens, client secrets) in `~/.config/fga/servers.yaml` without encryption. While the proxy binds to localhost by default, any process with filesystem access can read the configuration file. Users should follow best practices such as using short-lived tokens and restrictive file permissions (`0600`). Encryption at rest is a possible future enhancement, as is allowing secrets to be stored in the keychain with config fields like client_secret_cmd/api_token_cmd to specify commands for retrieving secrets.

## Alternatives

### Keep the existing hosted playground and fix CORS

The mixed-content and CORS issues affecting the existing `play.fga.dev` playground are structural. The playground calls browser `fetch()` from an HTTPS page to an HTTP local server. Modern browsers block this regardless of CORS headers. There is no fix that does not involve serving the playground from a non-HTTPS origin, which is not acceptable for a production web property. Continuing to maintain the existing closed-source codebase while these browser limitations make local-server connectivity increasingly broken is not a viable path.

### Ship the playground as a standalone Electron app

A native Electron application would avoid CORS entirely (Electron can make arbitrary HTTP calls) and would feel natural for a developer tool. However:

- Electron apps require OS-specific packaging, code signing, and distribution via app stores or direct download. This adds significant maintenance burden.
- Electron's rendering layer is Chromium, making it equivalent to a browser app anyway.
- The composable Web Component goal of embedding model editors and graphs in documentation would not be served by an Electron app.
- The shared config goal (one `~/.config/fga/servers.yaml` for CLI, playground, VS Code and IntelliJ) is served equally well by the `fga serve` approach without Electron's overhead.

### Build the UI with React

React is the dominant frontend framework and would give access to the largest ecosystem of component libraries. However:

- **Framework lock-in for consumers.** VS Code extension webviews are plain HTML/JavaScript contexts. React components cannot be used natively in them without bundling React into the webview's script. The same is true for any non-React documentation framework.
- **No native Web Component output.** React components are not Web Components. Embedding a React-based model editor in a Vue docs site or a plain HTML page requires wrapping it in an adapter, adding complexity for every consumer.
- **Version coupling.** Consumers of `@openfga/playground-components` would be forced to use a compatible React version. For Docusaurus that power our documentation site, it is usually tied to the React version Docusaurus needs, which would have the downstream effect of deciding what React version the Playground can support if we want to reuse it's components in the documentation. OpenFGA tooling should outlive individual React major versions; Lit's commitment to the browser standard avoids this problem.
- Lit is the thinnest abstraction over native Web Components. The resulting `<openfga-model-editor>` works in any HTML context, including VS Code webviews, React apps, Vue apps, and plain HTML documentation pages.

### Build the UI with Vue or Svelte

The same framework lock-in argument applies. Vue and Svelte both compile to framework-specific runtime code, not to native Web Components that any HTML context can consume without additional adapters. Lit's output is native.

### Use a TUI (terminal user interface)

A terminal UI (built with e.g. bubbletea or charm) would integrate naturally with the `fga` CLI and would not require running a browser. However:

- **Graph visualization** is an important feedback mechanism for understanding an authorization model. Terminals cannot render an interactive node-link graph in a way that is informative for non-trivial models.
- **No embeddability.** A TUI cannot be embedded in documentation or VS Code.
- **Accessibility.** The model graph is an inherently visual artifact. Communicating a graph structure in a terminal screen would require a fundamentally different design with unclear user experience.
- The existing `fga model get` and `fga tuple read` CLI commands already serve the text-only exploration use case. The Playground's value-add is the visual layer; a TUI would not provide that.
- A lot of folks using the Playground may not be as technical and would prefer a browser flow rather than a terminal based one.

### Use WebAssembly to run OpenFGA in the browser, no `fga serve`

A WASM build of OpenFGA would allow the playground to run entirely in the browser without needing `fga serve`. This would be the ideal experience for a fully hosted playground (e.g., launching `play.openfga.dev`), but it would not allow for the use-case of connecting to and managing remote authenticated services.

WASM is deferred for the following reasons:

- **Binary size.** Compiling the OpenFGA server to WASM produces a binary of 10–20 MB+. This is not acceptable for an initial page load, even with compression and lazy loading.
- **Implementation complexity.** OpenFGA uses a persistence layer (database) that must also be replaced with an in-memory WASM-compatible implementation. This is significant engineering work that goes beyond this RFC's scope.
- **Reduced need at v1.** A main  part of the target audience (developers who want to manage OpenFGA stores, in prod or dev) already have a server. The `fga serve` approach serves them well today.
- **Architecture readiness.** The `BackendAdapter` interface is explicitly designed to accommodate a WASM adapter in the future. When a WASM build of OpenFGA is available, it can be wrapped as a `WasmBackendAdapter` and plugged in without changing any component or core code.

That said, the hosted playground (WASM-only) can be a follow-up v2 goal.

### Use D3.js for graph visualization

D3 is the most flexible JavaScript visualization library. However:

- Graph layout with D3 requires manual implementation of a layout algorithm. Reproducing a quality hierarchical layout (like dagre) from scratch is substantial work.
- D3's API is low-level and imperative; integrating it with Lit's declarative rendering model requires careful lifecycle management.
- Cytoscape.js is purpose-built for graph visualization and includes built-in support for panning, zooming, click events, and keyboard navigation. The `cytoscape-dagre` plugin provides a deterministic hierarchical layout directly.
- Cytoscape is ~170 KB gzipped, comparable to D3's full bundle, and provides significantly more graph-specific functionality out of the box.

### Embed Monaco via the VS Code extension's webview panel directly

Monaco is the editor that powers VS Code. An alternative was to build the playground as a VS Code extension that renders its UI in a VS Code webview panel (which gives access to Monaco via VS Code's built-in API). However:

- This would make the playground unavailable to users who do not use VS Code.
- The Web Component approach delivers the same Monaco editor experience in any browser context.
- A VS Code extension is still a valid future consumer of the Web Components. It would embed `<openfga-model-editor>` in its webview, but building the components first gives us both possibiliities.

## Prior Art

- **`kubectl proxy`**: The pattern of a local CLI proxy that handles auth and CORS for a browser UI is well-established by `kubectl proxy` (for Kubernetes Dashboard) and `docker run -p` for local web UIs. `fga serve` follows the same pattern.

- **Vite, webpack-dev-server**: Both use the same localhost-only binding + Origin header validation pattern for their dev-mode proxies.

- **Auth0 FGA Dashboard**: The commercial Auth0 FGA product has its own playground UI embedded in the management console. The `BackendAdapter` interface could in theory allow the Auth0 FGA team to adopt `@openfga/playground-components` with a custom backend adapter.

- **Lit Web Components for tooling**: Google's Material Web, Adobe's Spectrum Web Components, and the FAST foundation (Microsoft) all use Lit or the native Web Component standard for component libraries that must work across framework boundaries. This validates the approach for a developer tooling context.

## Unresolved Questions

- **WASM timeline**: When will a WASM build of OpenFGA be available? The architecture is ready for it, but the timeline depends on OpenFGA server development priorities.

- **`fga serve` distribution**: Should `fga serve` be part of the main `fga` CLI binary or a separate install? And when is its release timeline?

- **Hosted playground**: Once WASM is available, we can launch `play.openfga.dev` as a WASM-only hosted playground. The new playground architecture supports this; the shell would use `WASMBackendAdapter` instead of `ProxyBackendAdapter`. This is out of scope for v1.

- **Session state**: The playground currently holds all state in nanostores atoms (in-memory). Refreshing the page reloads data from the server but loses any unsaved model edits, assertion definitions, and other local state. Persistence via `localStorage` or `sessionStorage` is not yet implemented, and is not, at this point in time, desired.
