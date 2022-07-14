# Meta
[meta]: #meta
- **Name**: Lookup API
- **Start Date**: 2022-07-14
- **Author(s)**: jon-whit
- **Status**: Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- **RFC Pull Request**: (leave blank)
- **Relevant Issues**:
  - https://github.com/openfga/openfga/issues/101
  - https://github.com/openfga/api/issues/10
- **Supersedes**: N/A

# Summary
[summary]: #summary

In some scenarios like UI filtering or auditing, developers need to answer queries like 

> list all documents that user:X can read

More specifically, given an object type, relation, and user, list all of the object ids of the given object type the user has the provided relationship with.

Depending on the use case, this problem can be solved with the existing Check API, as described in our [Search with Permissions](https://openfga.dev/docs/interacting/search-with-permissions) documentation, but in some cases it is more practical to have a dedicated API to serve this functionality.

The Lookup API addresses [Option 3 in the Search with Permissions](https://openfga.dev/docs/interacting/search-with-permissions#option-3-build-a-list-of-ids-then-search) documentation. It is specifically targeting the Search with Permissions use case for small object collections (on the order of thousands). For Search with Permissions on larger collections (tens of thousands and beyond) an OpenFGA client should build an external index via the ReadChanges (e.g. Watch) API or, in the future, via the OpenFGA Indexer.

# Definitions
[definitions]: #definitions
For more information/background on commonly used terms, please see the official [OpenFGA Concepts](https://openfga.dev/docs/concepts) page.

* [Search with Permissions](https://openfga.dev/docs/interacting/search-with-permissions) - Given a particular search filter (and optional sort order), what objects can the user acces?
* [Authorization Model](https://openfga.dev/docs/concepts#what-is-an-authorization-model)
* [Relationship Tuples](https://openfga.dev/docs/concepts#what-is-a-relationship-tuple)
* [gRPC Server Streaming RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc)
* [gRPC Unary RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc)

# Motivation
[motivation]: #motivation

- **Why should we do this?**
  It simplifies client integrations with OpenFGA for Search with Permissions and it adddress an ask from the community for an endpoint to list the objects a user has permissions for.

- **What use cases does it support?**
  Search with Permissions where the number of objects of a particular type the user could have access to is low (~ thousands), and the percentage of the total objects that the user can have access to is also low.

- **What is the expected outcome?**
  OpenFGA clients use the Lookup API to return a list of object ids and then the client uses the list to filter and order their object list in their applications (pagination on the filtered list is the clients responsibility). The expectation is that the client consumes the whole Lookup response before filtering client side, and does not try to partially filter based on a subset of the response from Lookup. Hence, it is not the goal of Lookup to be able to continue the Lookup process from a prior invocation.

# What it is
[what-it-is]: #what-it-is

The Lookup API will provide clients with the ability to query the object ids that a user has a particular relationship with. The list of object ids the Lookup API returns can then be used by a client to filter an object collection to only the objects that a user has a particular relationship with by taking the intersection of it with any client provided filters/ordering of the underlying dataset. This is described in more detail with [Option 3 of the Search with Permission](https://openfga.dev/docs/interacting/search-with-permissions#option-3-build-a-list-of-ids-then-search) document.

It will be provided as two different API endpoints: one will provide a streaming endpoint and the other will implement the endpoint without streaming. These two different endpoints could be used interchangably, but clients can choose which one may be better for their integration environment.

Consider a developer/client integrating with OpenFGA with the following authorization model:
```
type folder
    relations
        define viewer as self

type document
    relations
        define viewer as self or editor or viewer from parent
        define editor as self
        define parent as self
```
and the following relationship tuples:
```
document:doc1, viewer, bob
document:doc2, editor, bob
document:doc3, parent, folder:folder1
folder:folder1, viewer, bob
```

## StreamedLookup
StreamedLookup will implement a grpc [server streaming RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc) endpoint.

The endpoint will stream each object id the target user has the given relationship with as it is evaluated, and the server will continuing streaming results until either the lookup has been exhausted or the `streamed-lookup-deadline` has been reached (see [Configuration Changes](#configuration-changes)).

Sample grpcurl command:
```
grpcurl -plaintext \
  -d '{
           "store_id":"<store>", 
           "authorization_model_id":"<modelID>",
           "object_type":"document", 
           "relation":"viewer", 
           "user":"bob"
      }' \
  localhost:8081 openfga.v1.OpenFGAService/StreamedLookup

{
    "object_id": "doc1",
}
{
    "object_id": "doc2"
}
{
    "object_id": "doc3"
}
```

Sample curl command:
```
curl --request POST 'http://localhost:8080/stores/<storeID>/lookup:streamed' \
--header 'Content-Type: application/json' \
--data-raw '{
    "object_type": "document",
    "relation": "viewer",
    "user": "bob",
    "authorization_model_id": "<modelID>"
}'

{"result":{"object_id":"doc1"}}
{"result":{"object_id":"doc2"}}
{"result":{"object_id":"doc3"}}
```

## Lookup
Lookup will implement a grpc [unary RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc) endpoint.

The endpoint will return a list of the object ids the target user has the given relationship with. The size of the list that is returned will be determined by the `lookup-max-results` config (see [Configuration Changes](#configuration-changes)). If no maximum is provided by that config, then Lookup will attempt to accumulate all of the object ids before returning them. 
> ⚠️ without proper configuration of or use of the Lookup endpoint the server memory (and client memory) footprint could be profoundly impacted, because the list is accumulated in memory before it is returned.

Sample grpcurl command:
```
grpcurl -plaintext \
  -d '{
           "store_id":"<store>", 
           "authorization_model_id":"<modelID>",
           "object_type":"document", 
           "relation":"viewer", 
           "user":"bob"
      }' \
  localhost:8081 openfga.v1.OpenFGAService/Lookup

{
    "object_ids": [
        "doc1",
        "doc2",
        "doc3"
    ]
}
```

Sample curl command:
```
curl --request POST 'http://localhost:8080/stores/<storeID>/lookup' \
--header 'Content-Type: application/json' \
--data-raw '{
    "object_type": "document",
    "relation": "viewer",
    "user": "bob",
    "authorization_model_id": "<modelID>"
}'

{
    "object_ids": [
        "doc1",
        "doc2",
        "doc3"
    ]
}
```

The results from the Lookup query can then be used to filter a collection by taking the intersection of these object ids with objects returned from your application's query.

# How it Works
[how-it-works]: #how-it-works

The `StreamedLookup` endpoint will be (gRPC server stream) will be mapped to it's HTTP/json equivalent through the mapping provided by the implementation of the grpc-gateway project, which converts the stream into an [http/1.1 chunked transfer encoding](https://en.wikipedia.org/wiki/Chunked_transfer_encoding). Consumers of the streaming HTTP endpoint will be able to consume it using chunked-transfer encoding.

The `Lookup` endpoint (unary gRPC endpoint) will be mapped to it's HTTP/json equivalent through the mapping provided by the implementation of the [grpc-gateway](https://grpc-ecosystem.github.io/grpc-gateway/) project.

## API Changes
The following is the proposed patch diff of the protobuf API changes to [openfga/api](https://github.com/openfga/api):
```
diff --git a/openfga/v1/openfga_service.proto b/openfga/v1/openfga_service.proto
index 4140d56..d0056cb 100644
--- a/openfga/v1/openfga_service.proto
+++ b/openfga/v1/openfga_service.proto
@@ -661,6 +661,48 @@ service OpenFGAService {
       description: "Returns a paginated list of OpenFGA stores."
     };
   }
+
+  // StreamedLookup is a streaming variation of the Lookup API (see Lookup for more info).
+  rpc StreamedLookup(LookupRequest) returns (stream StreamedLookupResponse) {
+    option (google.api.http) = {
+      post: "/stores/{store_id}/lookup:stream",
+      body: "*"
+    };
+  }
+
+  // Lookup looks up the object ids for objects of the provided type that the given user
+  // has a specific relation with.
+  rpc Lookup(LookupRequest) returns (LookupResponse) {
+    option (google.api.http) = {
+      post: "/stores/{store_id}/lookup",
+      body: "*"
+    }
+  }
+}
+
+message LookupRequest {
+    string store_id  = 1 [
+      json_name = "store_id"
+    ];
+
+    string authorization_model_id = 2 [
+      json_name = "authorization_model_id"
+    ];
+
+    string object_type = 3 [
+      json_name = "object_type"
+    ];
+
+    string relation = 4;
+    string user = 5;
+}
+
+message LookupResponse {
+  repeated string object_ids = 1;
+}
+
+message StreamedLookupResponse {
+  string object_id = 1;
 }
```
The streaming variant accepts the same request body but differs in its response structure. For the streaming RPC a single `StreamedLookupResponse` is streamed back per result, whereas with the unary variant an array of object ids is accumulated and then returned.

The `/stores/{store_id}/lookup:streamed` HTTP path mapping for the streaming variant was inspired by [Google's HTTP mapping for Custom Methods](https://cloud.google.com/apis/design/custom_methods), but this is open for debate.

If anyone has any opinions or recommendations we'd love to hear them!

## Configuration Changes
- Add a `StreamedLookupDeadline` config param (`--streamed-lookup-deadline` flag)

  This will be used to define the maximum amount of time to stream Lookup results until the server stream is closed. This is to protect the server from misuse of the streaming endpoint. If this config is omitted all object ids will be streamed back to the client (regardless of how many there are).

- Add a `LookupMaxResults` config param (`--lookup-max-results` flag)

  This will be used to define the maximum list size of the list of object ids that are returned in the unary Lookup API endpoint. If this is omitted all object ids will be fetched (regardless of how many there are).
  
For production usage we should document recommendations for these configuration limits so as to protect the server from memory exhaustion or costly Lookup queries.

## Server Changes
* Add implementations of the API changes described in [Changes to the OpenFGA API](#changes-to-the-openfga-api) in the server. This includes implementations:
```
func (s *Server) StreamedLookup(
    req *openfgapb.LookupRequest,
    srv openfgapb.OpenFGAService_StreamedLookupServer,
) error {...}

func (s *Server) Lookup(
    ctx context.Context,
    req *openfgapb.LookupRequest,
) (*openfgapb.LookupResponse, error) {...}
```

### Storage Interface Changes

# Migration
[migration]: #migration

No migrations or breaking changes are necessary to support the Lookup API endpoints described herein.

# Drawbacks
[drawbacks]: #drawbacks

Potential arguments for not introducing the Lookup API include:
* Without proper limitations clients could impact the whole system by issuing Lookups on large collections.
* Increases the OpenFGA API surface for a very niche use case for Search with Permissions on small object collections.

# Alternatives
[alternatives]: #alternatives

An alternative version of Lookup could support both generic and specific queries such as:
> list all documents that user:bob can 'read' or 'write'

For example,
```
curl --request POST 'http://localhost:8080/stores/<storeID>/lookup' \
--header 'Content-Type: application/json' \
--data-raw '{
    "object_type": "document",
    "relations": ["read", "write"],
    "user": "bob",
    "authorization_model_id": "<modelID>"
}'
```
and

> list all relationships for documents that user:bob has

For example,
```
curl --request POST 'http://localhost:8080/stores/<storeID>/lookup' \
--header 'Content-Type: application/json' \
--data-raw '{
    "object_type": "document",
    "user": "bob",
    "authorization_model_id": "<modelID>"
}'
```

This alternative Lookup API definition is more generic and could be more generally applicable to specific cases where you want to know all of the objects (of a specific type) and the relationships to those objects a user has. However, the query cost of supporing a generic API endpoint like this could be quite expensive since you may have to evaluate all object and relation pairs for objects of a given type.

The stricter variant of Lookup that requires a single relationship protects the server a little more while giving control to the adopter to choose to how rate limit and throttle invocations of the Lookup endpoint. For these reasons it is recommended that Lookup will only serve queries such as those constrained to a specific object type, single relation, and single user.

# Prior Art
[prior-art]: #prior-art

## Other Zanzibar-inspired Implementations
* [Oso](https://www.osohq.com/) has a [`List` endpoint](https://cloud-docs.osohq.com/reference/client-apis/api-explorer#/default/post_list)
  The Oso List API requires `actor_type`, `actor_id`, `action`, and `resource_type` as inputs. So they limit the query to resources for a specific actor and action, which is similar to the primary proposal herein.

* [authzed/spicedb](https://github.com/authzed/spicedb) has a [`LookupResources` endpoint](https://app.swaggerhub.com/apis-docs/authzed/authzed/1.0#/PermissionsService/PermissionsService_LookupResources)
  The authzed/spicedb LookupResources API requires the `resourceObjectType`, `permission`, and `subject` (subject can be a userset such as 'group:group1#member'), so that is also similar to the primary proposal herein.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to be resolved before this gets merged?
- What parts of the design do you expect to be resolved through implementation of the feature?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
