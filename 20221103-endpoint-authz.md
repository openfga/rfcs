# Meta
[meta]: #meta
- Name: Endpoint authorization
- Start Date: 2022-11-08
- Author(s): jbrooks2-godaddy
- Status: Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- RFC Pull Request:
- Relevant Issues:
  <!-- List relevant Github issues here -->
- Supersedes: N/A

# Summary
[summary]: #summary

OpenFGA API operations have varying degrees of sensitivity. `Check` is relatively low-risk, but unrestricted access to write tuples or update authorization models is a security risk.

The endpoint authorization system addresses this concern by allowing OpenFGA platform operators to restrict authenticated access based on OIDC scopes or specific preshared keys on a global and per-endpoint basis.

# Definitions
[definitions]: #definitions

* [OIDC standard claims](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)
* [gRPC Unary RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc)

# Motivation
[motivation]: #motivation

## Why should we do this?
It allows OpenFGA operators to implement per-endpoint authorization without an additional proxy layer in front of OpenFGA.

## What use cases does it support?
OpenFGA operators who need to prevent all clients from full access to the OpenFGA APIs.

## What is the expected outcome?
OpenFGA operators can use the endpoint authorization configuration to optionally require a specific scope or subject in order to access an API (HTTP or gRPC) endpoint. This authorization will be opt-in; there will be no required migration for existing configurations.

# What it is
[what-it-is]: #what-it-is

## Overview

The endpoint authorization configuration extends the existing authentication config, allowing OpenFGA operators to require a specific subject, scope, or preshared key in order to access any given endpoint. To help reduce the verbosity of certain configurations we will also allow a top-level `global` authorization configuration that can be overridden on a per-endpoint basis.

First, we must identify what kind of callers are allowed. This will differ depending on the authentication method; `presharedkey` or `oidc`. For `presharedkey` we use the key itself. For `oidc` we can support either an allowed list of scopes or an allowed list of subjects.

Second, we must have a means to identify which routes need specific protections in the OpenFGA config. We must be careful not to use anything gRPC or HTTP specifc; this solution must work for either server configuration. The [`protobuf` API specification](https://github.com/openfga/api/blob/main/openfga/v1/openfga_service.proto#L14) is the common ground here, so let's use the RPC names (`ListStores`, `Read`, `Write`, `Check`, etc.) in the config.

For simplicity, permissions granted on an endpoint will be a union of all allowed subjects or scopes. If an endpoint is not present in the authorization configuration and no global authorization is configured, then all authentiated clients will be allowed access.

## Config structure
The means for identifying an authorized client is different for `preshared` and `oidc`, so there will be a method-specific configuration object defined for both of them and nested under their respective configuration object.

For both `preshared` and `oidc`, a key named `authz` will be added under the root. Under `authz` there will be two keys, `global` and `endpoints`. The `global` object will define the global authorization requirements, and the `endpoints` object will define the per-method overrides.

The `endpoints` key will be an object, the keys of which are an enum with the defined `protobuf` RPC methods as allowed values. The values of the RPC method keys and the `global` keys will differ depending on the authentication method.

`preshared` has one option, `keys`, which is a list of keys. These will be validated against the master list of allowed keys.

`oidc` will have two options: `scopes` and `subjects`. Either or both can be present, and each value is a list of strings representing the allowed scopes or subjects, respectively.

## Examples
A preshared key config that authorizes one key globally, and different keys on `Write` and `CreateStore`:

```yaml
authn:
  method: preshared
  preshared:
    keys:
      - cool-key-1
      - cool-key-2
    authz:
      global:
        keys:
          - cool-key-1
      endpoints:
        Write:
          keys:
            - cool-key-2
        CreateStore:
          keys:
            - cool-key-3
```

An `oidc` auth config that requires scopes to call only the `Write` endpoint:

```yaml
authn:
  method: oidc
  oidc:
    issuer: https://my-cool-oauth-server.com
    audience: my-audience
    authz:
      endpoints:
        Write:
          scopes:
            - openfga:write
            - openfga:read
```

A `oidc` config that authorizes one scope globally and two client subjects to call the `Write` endpoint (Note: a token with the scope `openfga:read` would not be able to call `Write`, unless its subject was one of the two authorized):
```yaml
authn:
  method: oidc
  oidc:
    issuer: https://my-cool-oauth-server.com
    audience: my-audience
    authz:
      global:
        scopes:
          - openfga:read
      endpoints:
        Write:
          subjects:
            - client-id-a
            - client-id-b
```

An `oidc` config that allows either subjects or scopes on an endpoint:

```yaml
authn:
  method: oidc
  oidc:
    issuer: https://my-cool-oauth-server.com
    audience: my-audience
    authz:
      endpoints:
        Write:
          subjects:
            - client-id-a
          scopes:
            - openfga:write
```

## API changes
The only API change is the addition of new authorization error codes. There are a [couple of unused codes](https://github.com/openfga/api/blob/main/openfga/v1/errors_ignore.proto#L7-L18) that might work:
```
auth_failed_invalid_subject
invalid_claims
```

But I propose that we define a new error that is authorization-specific:
```
auth_failed_unauthorized
```

# How it Works
[how-it-works]: #how-it-works

## Configuration definition
Adding the new configuration options is straightforward. Here is the updated `oidc` config:
```go
// AuthnConfig defines OpenFGA server configurations for authentication specific settings.
type AuthnConfig struct {
	// Method is the authentication method that should be enforced (e.g. 'none', 'preshared', 'oidc')
	Method                   string
	*AuthnOIDCConfig         `mapstructure:"oidc"`
	*AuthnPresharedKeyConfig `mapstructure:"preshared"`
}

// AuthnOIDCConfig defines configurations for the 'oidc' method of authentication.
type AuthnOIDCConfig struct {
	Issuer                string
	Audience              string
	*AuthnOIDCAuthzConfig `mapstructure:"authz"`
}

// AuthnOIDCAuthzConfig defines the authorization configuration for the 'oidc' method of authentication
type AuthnOIDCAuthzConfig struct {
	*AuthnOIDCAuthorization `mapstructure:"global"`
	Endpoints               AuthnOIDCProtectedEndpoints
}

// AuthnOIDCProtectedEndpoints defines the RPC endpoints that require authorization.
// The keys in ProtectedEndpoints must be valid openfgapb RPC methods
type AuthnOIDCProtectedEndpoints map[string]*AuthnOIDCAuthorization

// AuthnOIDCEndpointRestriction defines the means for authorizing something with OIDC.
// This could be a specific RPC endpoint or a higher level restriction.
type AuthnOIDCAuthorization struct {
	Subjects []string
	Scopes   []string
}
```

The options for `preshared` will be defined similarly. A brief note on `preshared` - right now, the `AuthClaims` are returned [without a subject](https://github.com/openfga/openfga/blob/main/server/authn/presharedkey/presharedkey.go#L37). To support authorizing specific preshared keys, this will need to be altered to return the key id as the subject in the claims. I believe that this is a reasonable semantic change.

## Configuration validation
The only validation to perform is on the keys indicating protected endpoints. When parsing the configuration, we can compare the provided keys to the RPC methods defined in the [OpenFGAService](https://github.com/openfga/api/blob/main/openfga/v1/openfga_service.proto#L14). If a key does not exist in the service, then the configuration will be rejected.

## Authorization evaluation
The [`Authenticator` interface](https://github.com/openfga/openfga/blob/main/server/authn/authn.go#L21-L28) will be extended, adding an `Authorize` function:

```go
type Authenticator interface {
	// Authenticate returns a nil error and the AuthClaims info (if available) if the subject is authenticated or a
	// non-nil error with an appropriate error cause otherwise.
	Authenticate(requestContext context.Context) (*AuthClaims, error)

	// Authorize returns a nil error if the subject is authorized or a non-nil error with an appropriate error cause
	// otherwise. It requires that the context has been augmented with AuthClaims
	Authorize(requestContext context.Context, fullPath string) error

	// Close Cleans up the authenticator.
	Close()
}
```

As the comment indicates, the `Authorize` function will rely on the `Authenticate` function augmenting the request context with the claims extracted from the `Authorization` header. Additionally, it requires the full method path to be supplied in order to compare against the configured protected endpoints.

The [`AuthFunc` middleware](https://github.com/openfga/openfga/blob/main/internal/middleware/authn.go#L10-L19) will be augmented to support performing an authorization check. In order to access the full request method, it must be constructed the `grpc.UnaryServerInterceptor` directly instead of via `github.com/grpc-ecosystem/go-grpc-middleware/auth`. The middleware will first perform the authentication check with `Authenticate` and augment the request context. If there are no errors, it will then invoke `Authorize`.

Inside of `Authorize`, evaluation is performed as follows:
1. Extract the method name from the fullPath and lowercase.
2. If the method name exists in the `ProtectedEndpoints` config:
   1. If either the scope or the subject exist in the protected endpoint config for the method name, then return `nil`.
3. If there is a global config:
   1. If either the scope or the subject exist in the global config, then return `nil`.
4. Return an authorization error.

# Migration
[migration]: #migration

No migration is needed; support for endpoint authorization will be completely opt-in.

# Drawbacks
[drawbacks]: #drawbacks

* If the implementation is simple, it will not support all authorization use-cases (i.e. combinations of allow and deny).
* If the implementation allows for more complex authorization schemes, it will be more difficult to configure and more prone to implementation bugs.

# Alternatives
[alternatives]: #alternatives

## Tweaks to this design
There are a couple of additions/deletions to this design that make it easier to define certain authorization schemas at the expense of additional complexity.

### Don't allow global scopes
Originally, this design did not include the global scope configs. It does add a small amount of complexity, but I think that usage will be common enough that including it as an option is worth it. We could require explicit authorizaiton on every endpoint instead.

### Read/write endpoint classification
It may be a reasonable assumption that most clients may want to create distinct restrictions on read and write endpoints. We could classify each endpoint as one of the two and expose a config option for each, again allowing the same variety of subject authorization definitions.

```yaml
authn:
  method: oidc
  oidc:
    issuer: https://my-cool-oauth-server.com
    audience: my-audience
    read_authz:
      subjects:
        - client-id-a
      scopes:
        - openfga:read
    write_authz:
      subjects:
        - client-id-a
      scopes:
        - openfga:write
```

I believe that this approach is too prescriptive; we should leave this in the hands of the operators even if it requires a bit more verbosity in the configuration.

## Other approaches
### OpenFGA-drive authorization
Given that OpenFGA is an authorization system, could we use it to answer the authorization question of "Can this principal take this action on the system?" The key challenge here is forming the question. Can we expose configurations in a way that allow enough flexibility for the operator? An OpenFGA authorization question takes the form:
```
Check(user, relation, object_type:object_id)
```

When a client makes a request to an OpenFGA endpoint we must fill in each of the parameters with a value. The `user` is simple - the API client making the request. This will differ depending on the authentication method, but I don't think that this needs configuration - any given authentication method will have a single subject/principal, so that will be extracted and inserted as the `user` in the authorization check.

`relation` and `obejct_type:object_id` are trickier, as these depend on the authorization model being used. I think that OpenFGA should avoid prescribing an authorization model. Instead, we can use the configuration to let operators tell the system exactly how to structure the authorization question. As with the simple system, we can allow for simple global authorization combined with fine-grained per-endpoint overrides. For example:

Simple global access. Any subject that has the relation `can_access` on instance `global` of object type `openfga_api` can access any API:
```yaml
authz:
  store_id: aaa-bbb-ccc
  global:
    relation: can_access
    object_type: openfga_api
    object_id: global
```
On any call:
```
Check(client-id, can_access, openfga_api:global)
```

Global access with an override. The relation `can_write` is required to call the `Write` endpoint:
```yaml
authz:
  store_id: aaa-bbb-ccc
  global:
    relation: can_access
    object_type: openfga_api
    object_id: global
  endpoints:
    Write:
      relation: can_write
      object_type: openfga_api
      object_id: global
```
On a `Write` call:
```
Check(client-id, can_write, openfga_api:global)
```
On any other call:
```
Check(client-id, can_access, openfga_api:global)
```

The `object_id` in all of these is `global`. It could be anything, really, as we are not authorizing specific instances of the openfga API. We could extend this system and put that to use - it would be nice to further restrict clients dynamically based on the content of the request:
- Restrict which clients can take any operation on a given store
- Restrict which clients can write a given authorization model
- Restrict which clients can write tuples to a given object

We have the protobuf reference available, so we could reference fields from the `Request` objects in these definitions with special syntax, e.g. `$store_id`:
```yaml
authz:
  store_id: aaa-bbb-ccc
  global:
    relation: can_access
    object_type: openfga_api
    object_id: global
  endpoints:
    Write:
      relation: can_write
      object_type: openfga_api_store
      object_id: $store_id
```
During a `Write` request from subject `subject-1` to store `store-1` the authorization code would parse out the store id from the request and make the following `Check`:
```
Check(subject-1, can_write, openfga_api_store:store-1)
```

This is easy enough for simple top-level attributes like `store_id` or `authorization_model_id`, but becomes more challenging if we want to do something like restrict access to write tuples to a particular object type. A `Write` request can contain multiple tuples in the `writes` and `deletes` field, it would be awkward to use a generic syntax in the configuration to reference the `object_type` in the tuples.

There aren't that many attributes to restrict. Instead of a generic syntax that references the protobuf request messages, we could hard-code the attributes on which we allow dynamic authorization restrictions:
- store_id
- authorization_model_id
- object_type
- contextual_tuple_object_type
- (maybe more...)

These can be validated in the configuration to ensure that they are only applied to the appropriate endpoints. To preserve the ability to authorize a client on an endpoint globally we can support two object id fields, `static_object_id` and `dynamic_object_id`. For example:

```yaml
authz:
  store_id: aaa-bbb-ccc
  global:
    relation: can_access
    object_type: openfga_api
    object_id: global
  endpoints:
    Write:
      relation: can_write
      object_type: openfga_api_write
      static_object_id: global
      dynamic_object_id: object_type
```

This configuration has the following properties:
- On a request to `Write`, at least two `Check` calls are made:
  - `Check(client-id, can_write, openfga_api_write:global)`.
    - If this check call succeeds, then the request is authorized.
  - `Check(client-id, can_write, openfga_api_write:$object_id)` for each distinct `object_id` included in the `Write` request
    - If _all_ of these check calls succeed, then the request is authorized.
- On any other request, a single `Check` call is made:
  - `Check(client-id, can_access, openfga_apoi:global)`
    - If this check call succeeds, then the request is authorized.

This approach is powerful but has some drawbacks:
- Multiple `Check` calls to authorize a request could be expensive and slow.
  - This is controllable by the operator, though, and likely a reasonable tradeoff - it's ok for `Write` and similar calls to be slower. It's unlikely that `Check` would have a complex dynamic authorization scheme
- It is more complicated to implement and configure.
- There is a bootstrapping problem - you must manually insert tuples or iteratively change the authorization scheme to allow the first writes.

# Prior Art
[prior-art]: #prior-art

Neither Ory Keto nor SpiceDB support any form of endpoint-based authorization. Ory Keto requires a trusted gateway for any form of authN/authZ, and SpiceDB supports preshared keys for authentication but no authorization past that.

Generally, authorizing OIDC clients based on scopes or subjects is a fairly standard practice. 

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

## What parts of the design do you expect to be resolved before this gets merged?
The level of configuration to support:
- Require enumerating authorization on all endpoints.
- Allow global authorization.
- Segregate endpoints into read/write or levels of sensitivity.

## What parts of the design do you expect to be resolved through implementation of the feature?
None, the implemementation is straightforward based on the designs outlined in this document.

## What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
Support for other authentication methods and support for more complex authorization schemes.
