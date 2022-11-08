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
### OpenFGA-driven authorization
Given that OpenFGA is an authorization system, we could prescribe an authorization model within OpenFGA that models access to the various OpenFGA API operations. Something along the lines of:
```
type openfga_api
  relations
    define full_access as self
    define can_create_store as self or full_access
    define can_write_tuple as self or full_access
    ...etc
```

The OpenFGA authorization code would then make a `Check` call on each request to authorize a client. While dogfooding is appealing at its surface, this approach has some drawbacks:
- It requires setup outside of just writing a configuration file.
- It couples the authorization system to other parts of OpenFGA.
- There is a chicken-and-egg problem; you cannot protect the system prior to starting it up.
- OpenFGA would be prescribing an authorization model that might not work for the operator

I think that the benefits do not outweigh the extra complexity.

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
