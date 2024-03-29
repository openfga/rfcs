# Add Type Restrictions to the JSON Syntax

## Meta

- **Name**: Add Type Restrictions to the JSON Syntax
- **Start Date**: 2022-08-31
- **Author(s)**: [rhamzeh](https://github.com/rhamzeh)
- **Status**: Approved
- **RFC Pull Request**: <https://github.com/openfga/rfcs/pull/7>
- **Relevant Issues**:
  - <https://github.com/openfga/rfcs/pull/3>
  - <https://github.com/openfga/rfcs/pull/4>
  - <https://github.com/openfga/rfcs/pull/8>
  - <https://github.com/openfga/api/pull/27>
  - <https://github.com/openfga/syntax-transformer/pull/47>
- **Supersedes**: N/A

## Summary

Allow users to indicate in their store's authorization model what types of objects can have a particular relation to an object type.

## Definitions

- [OpenFGA JSON Syntax](https://openfga.dev/docs/configuration-language)
- [OpenFGA DSL](https://openfga.dev/docs/configuration-language)
- [ListObjects Endpoint](https://openfga.dev/api/service#/Relationship%20Queries/ListObjects)
- [What is a user?](https://openfga.dev/docs/concepts#what-is-a-user)
- Floating User ID: A user identifier without a type (e.g. `anne`, `4` or `4179af14-f0c0-4930-88fd-5570c7bf6f59`)
- Reverse Expansion: Normally [expand](https://openfga.dev/docs/interacting/relationship-queries#expand) takes an object and relation and returns the first leaves of users who are related. Reverse expand would be the opposite, taking a user and a relation and returning the objects which are related.
- Representing public access: [`*`](https://openfga.dev/docs/concepts#how-do-i-represent-everyone)

## Motivation

### Why should we do this?

#### Optimize ListObjects

In order to implement some optimizations to the ListObjects endpoint mentioned [ListObjects RFC](https://github.com/openfga/rfcs/blob/main/20220714-listObjects-api.md), we should be able to traverse the relationship graph in reverse. We not only need to understand how objects are related to other objects, but also what type of objects can be related.

#### Improve the Developer Experience

At the moment, our authorization model does not allow users to indicate what the type they expect on a relation to be, leaving it ambiguous and potentially a cause of errors, especially when someone is reading the authorization model later without much context and where the original authors are not available. It is hard for someone to grok from first glance that a repository owner should be an organization or a user. Adding type restrictions allows developers to have an easier time in interpreting the model.

### What is the expected outcome?

- Using the extra type information we get from this change, we will be able to more easily traverse the graph in reverse, thus allowing us to provide a solution for reverse expansion and optimizing the ListObjects endpoint.
- Users reading an authorization model will be able to better interpret its meaning (for example by understanding that repository owners must be either users or group members).

## What it is?

### Adding Type Restrictions

This RFC proposes an update to the OpenFGA JSON syntax requiring users to indicate all the types of users that could be **directly related** to an object of a certain type through a particular relation.

For example, our current syntax allows expressing the following:

- documents have parents
- repositories have owners

But cannot express restrictions such as:

- parents of a document have to be objects of type folder
- owners of a repository have to be of type user or a userset of group members

We introduce a `metadata` entry in the type definition would look like:

```typescript
type Metadata = {
  relations?: Record<string, RelationMetadata>;
};

type RelationMetadata = {
  directly_related_user_types?: DirectlyRelatedUserType[];
};

type DirectlyRelatedUserType = {
  type: string;
  relation?: string;
  wildcard?: Record<string, never>;
};
```

The following is an example of the changes we are proposing, an explanation of the additions will follow:

```json5
{
   "type_definitions": [
      {
         "type": "user",
         "relations": {}
      },
      {
         "type": "employee",
         "relations": {}
      },
      {
         "type": "group",
         "relations": {
            "parent": {
               "this": {}
            },
            "member": {
               "this": {}
            },
            "guest": {
               "this": {}
            }
         },
         "metadata": {
            "relations": {
               "parent": {
                  "directly_related_user_types": [
                     {
                        // Groups can be parents of other groups
                        "type": "group"
                     }
                  ]
               },
               "member": {
                  "directly_related_user_types": [
                     {
                        // An object of type employee can be a member of the group
                        "type": "employee"
                     },
                     {
                        // A set of objects who are related to a group as members can be made members of a group (e.g. members of the admin group are members of the security group)
                        "type": "group",
                        "relation": "member"
                     }
                  ]
               },
               "guest": {
                  "directly_related_user_types": [
                     {
                        // An object of type user can be a guest of the group
                        "type": "user"
                     },
                     {
                        // A group can be made public (have all objects of type user be guests of it)
                        "type": "user",
                        "wildcard": {}
                     },
                     {
                        // An object of type employee can be a guest of the group
                        "type": "employee"
                     },
                     {
                        // A set of objects who are related to a group as members can be made guests of a group (e.g. members of the admin group are guests of the employee group)
                        "type": "group",
                        "relation": "member"
                     }
                  ]
               }
            }
         }
      }
   ]
}
```

In the type metadata, add a `directly_related_user_types` array to each relation to indicate what types of objects can be directly related to the relation. It will be an array of objects, each object must be composed of:
- a type: indicates that objects of this type can be related
- a type and an optional relation: indicates that sets of objects who are related to that type as that relation can be related
- a type and an empty wildcard object: indicates that the special set of all objects of these types can be related

In the above example:
- having the following `"parent": { "directly_related_user_types": [{ "type": "group" }] }` in the `group` type definition indicates that only objects of type `group` can be directly related to a `group` as `parent`
- having the following `"member": { "directly_related_user_types": [{ "type": "user" }, { "type": "group", "relation": "member" }] }` in the `group` type definition indicates that only objects of type `user` or usersets of type `group` and relation `member` can be directly related to a `group` as `member`
- having the following `"guest": { "directly_related_user_types": [{ "type": "user" }, { "type": "user", "wildcard": {} }, { "type": "employee" }] }` in the `group` type definition indicates that the `user:*` syntax is allowed and that when present, it means all objects of type `user` can be directly related to a `group` as `guest`

This will affect only relations that are [directly related](https://openfga.dev/docs/modeling/building-blocks/direct-relationships) (as in they are considered "assignable" and have [the direct relationship keyword ("this")](https://openfga.dev/docs/configuration-language#the-direct-relationship-keyword) in their relation definition).

For relation definitions that:

- have the direct relationship keyword (this): `directly_related_user_types` must be present and MUST be an non-empty array containing at least one valid type restriction
- do not have the direct relationship keyword: `directly_related_user_types` can optionally be present, but if it is, it MUST be an empty array

> Note: This syntax does not offer a way to enforce restrictions based on what the userset of group members resolves to (for example, group member can be a user, an employee or another userset). Later on, tooling can help visually indicate this to the user inline.

To prevent breaking changes, this will be done by appending a new field called `metadata` -> `relations` to each type definition. Each relation will be a key under this that is a map that contains a field called `directly_related_user_types` that contains what type or type and relation combination could be directly related to it and another called `allow_public` that defines whether the `*` is valid.

### Change in the concept of everyone [Breaking]

As part of adding type restriction, we decided to change how the concept of everyone (previously represented as `*`) to allow it to also be typed.

- In schema version 1.1, `*` is no longer supported (along with floating user_ids). If there are existing relationship tuples in the system that are either `*` or a floating user ID they will be ignored when evaluating using a 1.1 schema version and writes including them to a 1.1 schema version will fail validation
- To represent all objects of a certain type, the following syntax is introduced: `${type}:*`. For example, `employee:*` in the user field of a relationship tuple means all objects of type `employee`. Note that this syntax can only be used in the user field and not in the object field. Note that `employee:*` will match any object of type `employee`, including those not in existing relationship tuples.
   In order to prevent confusion, we chose to disallow objects of format `employee:*` on v1.0 models going forward, as that will be interpreted as an object of type `employee` and id `*` on the previous version and every object of type `employee` on the v1.1 model

### Adding a Schema Version Field

In order to make sure we can easily parse the model across updates (and this will be especially true when breaking changes are introduced), we are proposing to add a version to the model.

This version will be called the `schema_version` in order not be confused with an update to the authorization model itself (frequently referenced as a new authorization model version). It will be of the form "x.y", where both `x` and `y` are non-negative integers. `x` will start from `1` instead of `0`, so the initial version of the authorization model will be `1.0` and this RFC once implemented will introduce `1.1`.

This field is optional, and when missing will be interpreted as being `1.0`. The schema version can only be one of the existing versions ("1.0" and "1.1" at the time of this RFC).

```json
{
  "schema_version": "1.1",
  "type_definitions": [ ... ]
}
```

### Examples

You can see some examples of how authorization models will need to change with this new proposed extension in [this PR](https://github.com/openfga/sample-stores/pull/4).

<details>
<summary>Entitlements</summary>

[Entitlements Sample Store](https://github.com/openfga/sample-stores/blob/feat/add-type-restrictions/stores/entitlements)

```diff
{
+ "schema_version": "1.1",
  "type_definitions": [
+   { "type": "user" },
    {
      "type": "plan",
      "relations": {
        "subscriber": {
          "this": {}
        },
        "subscriber_member": {
          "tupleToUserset": {
            "tupleset": {
              "object": "",
              "relation": "subscriber"
            },
            "computedUserset": {
              "object": "",
              "relation": "member"
            }
          }
        }
+     },
+     "metadata": {
+       "relations": {
+         "subscriber": {
+           "directly_related_user_types": [{
+             "type": "organization"
+           }],
+         },
+         "subscriber_member": {
+           "directly_related_user_types": []
+         }
+       }
      }
    },
    {
      "type": "organization",
      "relations": {
        "member": {
          "this": {}
        }
+     },
+     "metadata": {
+       "relations": {
+         "member": {
+           "directly_related_user_types": [{
+             "type": "user"
+           }]
+        }
      }
    },
    {
      "type": "feature",
      "relations": {
        "access": {
          "tupleToUserset": {
            "tupleset": {
              "object": "",
              "relation": "associated_plan"
            },
            "computedUserset": {
              "object": "",
              "relation": "subscriber_member"
            }
          }
        },
        "associated_plan": {
          "this": {}
        }
+     },
+     "metadata": {
+       "relations": {
+         "associated_plan": {
+           "directly_related_user_types": [{
+             "type": "plan"
+           }]
+         },
+         "access": {
+           "directly_related_user_types": []
+         }
+       }
      }
    }
  ]
}
```

</details>

<details>
<summary>Expenses</summary>

[Expenses Sample Store](https://github.com/openfga/sample-stores/blob/feat/add-type-restrictions/stores/expenses)

```diff
{
+ "schema_version": "1.1",
  "type_definitions": [
+   { "type": "user" },
    {
      "type": "report",
      "relations": {
        "approver": {
          "tupleToUserset": {
            "tupleset": {
              "object": "",
              "relation": "submitter"
            },
            "computedUserset": {
              "object": "",
              "relation": "manager"
            }
          }
        },
        "submitter": {
          "this": {}
        }
+     },
+     "metadata": {
+       "relations": {
+         "approver": {
+           "directly_related_user_types": []
+         },
+         "submitter": {
+           "directly_related_user_types": [{
+               "type": "user"
+           }]
+         }
+       }
      }
    },
    {
      "type": "employee",
      "relations": {
        "manager": {
          "union": {
            "child": [
              {
                "this": {}
              },
              {
                "tupleToUserset": {
                  "tupleset": {
                    "object": "",
                    "relation": "manager"
                  },
                  "computedUserset": {
                    "object": "",
                    "relation": "manager"
                  }
                }
              }
            ]
          }
        }
+      },
+      "metadata": {
+        "relations": {
+          "manager": {
+            "directly_related_user_types": [{
+                "type": "user"
+            }]
+          }
+       }
      }
    }
  ]
}
```

</details>

## How it Works

### API Changes

#### Validating Authorization Model Writes

When writing new models in the new syntax, we need to validate:

1. That the directly related user types array exists, and
   1. is empty when the relationship definition does not allow this (direct relationships)
   2. is non-empty when the relationship definition does allow this (direct relationships)
2. When directly related user types are passed, we need to check that:
   1. the type is an existing type in the system
   2. the relation is either empty or exists on the type
   3. no duplicates are present

```javascript
{ "schema_version": "1.1",
  "type_definitions": [
  { "type": "user", "relations": {} },
  { "type": "group",
    "relations": {
      "relation-1": { "this": {} },
      "relation-2": { "this": {} },
      "relation-3": { "this": {} },
      "relation-4": { "this": {} },
      "relation-5": { "this": {} },
      "relation-6": { "computedUserset": { "object": "", "relation": "relation-1"} },
      "relation-7": { "computedUserset": { "object": "", "relation": "relation-1"} },
      "relation-8": { "this": {} },
      "relation-9": { "this": {} },
      "relation-10": { "this": {} },
    },
    "metadata": {
      "relations": {
        "relation-1": { "directly_related_user_types": [{ "type": "user" }] }, // valid, user exists as a type
        "relation-2": { "directly_related_user_types": [{ "type": "group", "relation": "relation-1" }] }, // valid, group exists as a type, and relation-1 exists on that type
        "relation-3": { "directly_related_user_types": [] }, // invalid, relation-3 allows direct relationships, but no directly related user types are set
        "relation-4": { "directly_related_user_types": [{ "type": "group", "relation": "relation-0" }] }, // invalid, group exists, but relation-0 does not exist on that type
        "relation-5": { "directly_related_user_types": [{ "type": "user" }, { "type": "user" }] }, // invalid, duplicate found
        "relation-6": { "directly_related_user_types": [{ "type": "user" }] }, // invalid, relation-6 does not allow direct relationships (no `this` in the relation definition)
        "relation-7": { "directly_related_user_types": [] }, // valid, relation-7 does not allow direct relationships, so no elements are expected
        "relation-8": { "directly_related_user_types": [{ "type": "group", "wildcard": {} }] }, // valid
        "relation-9": { "directly_related_user_types": [{ "wildcard": {} }] }, // invalid, each directly_related_user_type must have a relation
        "relation-10": { "directly_related_user_types": [{ "type": "group", "relation": "relation-1", "wildcard": {} }] }, // invalid, a directly_related_user_type cannot have both a relation and a wildcard
      }
    }
  }
] }
```

#### Validating Tuple Writes

We'll use the following example authorization model:

```json
{ "schema_version": "1.1",
  "type_definitions": [
  { "type": "user", "relations": {} },
  { "type": "group",
    "relations": {
      "parent": { "this": {} },
      "member": { "this": {} },
    },
    "metadata": {
      "relations": {
        "parent": { "directly_related_user_types": [{ "type": "group" }] },
        "member": { "directly_related_user_types": [{ "type": "user", "wildcard": {} }, { "type": "group", "relation": "member" }, { "type": "employee" }] }
      }
    }
  },
] }
```

On write, we need to validate that:

1. If the user is an object:
   1. The type of the user should be in the list of directly related user types on the relation of the type of the object
   - `write(user=user:1, member, group:1)`; valid, the `user` type is in the list of directly related user types for the `member` relation of the `group` type
   - `write(user=group:2, parent, group:1)`; valid, the `group` type is in the list of directly related user types for the `parent` relation of the `group` type
   - `write(user=group:2, member, group:1)`; invalid, the `group` type is not in the list of directly related user types for the `member` relation of the `group` type
   - `write(user=user:1, parent, group:1)`; invalid, the `user` type is not in the list of directly related user types for the `parent` relation of the `group` type

2. If the user is a userset:
   - `write(user=group:2#member, member, group:1)`; valid, the `(type=group, relation=member)` is in the list of directly related user types for the `member` relation of the `group` type
   - `write(user=group:2#member, parent, group:1)`; invalid, the `(type=group, relation=member)` is not in the list of directly related user types for the `parent` relation of the `group` type
   - `write(user=group:2#parent, member, group:1)`; invalid, the `(type=group, relation=parent)` is not in the list of directly related user types for the `member` relation of the `group` type
   - `write(user=group:2#parent, parent, group:1)`; invalid, the `(type=group, relation=parent)` is not in the list of directly related user types for the `parent` relation of the `group` type

3. If the user is a typed wildcard `${type}:*`:
   - `write(user=user:*, member, group:1)`; valid, it will mean all objects that are of type `user` are related to `group:1` as `member`
   - `write(user=*, member, group:1)`; invalid, v1.1 schema version dropped support for the `*` user syntax
   - `write(user=user*, can_view, group:1)`; invalid, the `can_view` relation on the `group` type does not allow direct relationship tuples (no `this` in the relation definition).
   - `write(user=group:*, parent, group:1)`; invalid, `(type=group, wildcard={})` is not in the `directly_related_user_types` array
   - `write(user=group:1#parent, parent, group:1)`; invalid, the type restrictions on the `parent` relation on the `group` type does not allow `(type=group, relation=parent)`

> Note: Any tuples that are considered invalid on write according to a certain model should also be ignored when evaluating the graph based on that model even if they already exist in the database.

#### Updating ListObjects to Respect Type Restrictions

[ListObjects](https://openfga.dev/api/service#/Relationship%20Queries/ListObjects) will be the first of the Relationship Query endpoints to take advantage of this new functionality. Our hope is that we can start building a more performant ListObjects endpoint using the new type restrictions.

#### Updating Expand to Respect Type Restrictions

Once ListObjects has been implemented and tested, [Expand](https://openfga.dev/api/service#/Relationship%20Queries/Expand) will need to be updated to ignore tuples that do not match the directly related user types in the authorization model.

#### Updating Check to Respect Type Restrictions

[Check](https://openfga.dev/api/service#/Relationship%20Queries/Check) will be the final phase and should be undertaken only once we are completely confident of how ListObjects and Expand are performing with the new functionality. Check is the core of OpenFGA, and we should be diligent in making sure it does not break and that any changes are properly communicated.

When the time comes, check will be updated to ignore relationship tuples in the database that do not match the directly related user types in the authorization model.

## Migration

### Migrating existing authorization models

Existing models will not be migrated - new models will be required to use types (with an optional grace period). Authorization models that do not make use of types will not be able to use the optimized ListObjects endpoint and will fallback to the existing brute force implementation.

### Migrating existing tuples

Due to the users in the tuples now required to have types in order to enforce the restrictions, existing tuples with users as floating user identifiers with no types (e.g. `anne`) will no longer be valid when types are added. These tuples will not be removed from the system, and will still be valid while the legacy authorization model is supported, but will be ignored during evaluation on newer authorization models with type restrictions in place.

This will need to be communicated to developers so that they can migrate accordingly by:

1. Introducing a `user` type to the model
2. Reading all exiting relationship tuples to find those with a floatig user id (user that has only an identifier and no type)
3. Writing a copy of that tuple but with the `user` type
4. Migrating their app code to perform checks using that type

One option for developers administrating an OpenFGA installation could be a script to check the DB (look for tuples in the DB that do not have `user_type`), and print the offending store IDs.

Automatic unassisted tuple migration will not be feasible because it is not possible to know what type end users will want to use for each offending tuple.

### Affected Modules

This change will affect repositories across the board. It will entail changes to the [protobufs](https://github.com/openfga/api), the [openfga core](https://github.com/openfga/openfga), the DSL, the [syntax transformer](https://github.com/openfga/syntax-transformer), the [SDKs](https://github.com/openfga/sdk-generator), the FGA Playground, the [sample stores](https://github.com/openfga/sample-stores) and the [documentation](https://github.com/openfga/openfga.dev).

#### Language, API and SDKs

For the scope of this RFC, we are proposing that the initial phase be backwards compatible and not a breaking change. This will allow users on previous versions to keep using them during a grace period while keeping changes to the public surface of the API and SDKs to a minimum.

#### DSL

As a lot of developers will now have to include a user type and it may not have any relations on it. An update to the DSL needs to happen to support types with no relations: (completed via openfga/syntax-transformer#47)

```python
type user
```

An update to the DSL needs to be drafted to support the inclusion of the type restrictions, this should come in a [later RFC](https://github.com/openfga/rfcs/blob/type-restriction-dsl/20221012-add-type-restrictions-to-dsl-syntax.md).

#### Syntax Transformer and Playground

Syntax transformer will need to be updated to support both the new JSON syntax and the new DSL it maps to, as well as all the necessary validations. (Completed via [syntax-transformer v0.0.8](https://github.com/openfga/syntax-transformer/releases/tag/v0.0.8))

### Roadmap

1. ~~[This RFC](https://github.com/openfga/rfcs/pull/7) is drafted~~
2. ~~The [api protobufs](https://github.com/openfga/api/blob/f10bb663ad633df8c6f7cdcfc7280b959ab47293/openfga/v1/authzmodel.proto#L52) are updated to allow types with no relations~~
3. ~~The [openfga/api#27](https://github.com/openfga/api/pull/27) PR introduces the new fields into the proto-files~~
4. ~~The DSL & [openfga/syntax-transformer](https://github.com/openfga/syntax-transformer) are updated to allow empty user type~~ (completed via openfga/syntax-transformer#47)
5. ~~[openfga/openfga.dev](https://github.com/openfga/openfga.dev) is updated to use the user type across the board~~
6. [openfga/openfga](https://github.com/openfga/openfga)
   1. ~~Validation is added to prevent writing models with invalid type restrictions (e.g. restricting to a type that does not exist)~~
   2. ~~Validation is added to prevent writing tuples that do not match the type restrictions~~
   3. ListObjects implementation is updated to take the type restrictions into consideration
7. ~~An RFC for the updated DSL that supports type restrictions is [drafted](https://github.com/openfga/rfcs/blob/type-restriction-dsl/20221012-add-type-restrictions-to-dsl-syntax.md)~~
8. ~~[openfga/syntax-transformer](https://github.com/openfga/syntax-transformer) is updated with support for the new DSL and JSON syntax~~
9. ~~Playground is updated with the latest syntax-transformer changes~~
10. ~~[openfga/sdk-generator](https://github.com/openfga/sdk-generator) is updated to reflect the changes in the proto files~~
11. [openfga/sample-stores](https://github.com/openfga/sample-stores) is updated with the type restrictions
12. [openfga/openfga.dev](https://github.com/openfga/openfga.dev) is updated to include type restrictions in the documentation
13. ~~Expand and Check implementations are updated to take the type restrictions into consideration~~

## Drawbacks

- Floating user_ids with no type can no longer be supported (as adding type restrictions on direct relations requires that objects have a type)
- The `*` syntax with no type is no longer be supported
- The JSON syntax now contains duplicate fields for each relation (the additional one being in metadata containing directly related user types)

## Alternatives

### Location of the Directly Related User Types Restrictions

#### Require the resolved type to be indicated instead of the direct types

```json
{ "type_definitions": [
  { "type": "user", "relations": {} },
  { "type": "group",
    "relations": {
      "parent": { "this": {} },
      "member": { "this": {} },
    },
    "metadata": {
      "relations": {
        "parent": { "directly_related_user_types": ["group"] },
        "member": { "directly_related_user_types": ["user"] }
      }
    }
  },
] }
```

Allowing users to set the types that a relation can resolve to, can help us build tooling for them that would ensure that their userset rewrites are valid.

For example, consider the following model (in psuedocode). If someone writing the authorization model tries to set this as parent, we can raise an error because we know that parent resolves to a folder, while editor needs to resolve to a user. Which would improve the developer experience and help us guide the users to resolve issues in their models.

```javascript
{ "type_definitions": [
  { "type": "user", "relations": {} },
  { "type": "folder" },
  { "type": "document",
    "relations": {
      "parent": { 
        "directly_related_user_types": ["folder"],
        "this": {}
      },
      "editor": {
        "directly_related_user_types": ["user"],
        "computedUserset": {
          "object": "",
          "relation": "parent" // we can detect an error, because parent resolves to folder while editor expects user
        }
      },
    }
  },
] }
```

This alternative would have been our choice had we been optimizing for developer experience instead of for resolving the ReverseExpand/ListObjects use-case. However, because it does not help us narrow down the address space when traversing the graph in reverse, it was deemed insufficient to meet our needs.

#### Add the Directly Related User Types Directly in the Main Relation

It would have been cleaner to introduce the directly related user types into the relation object like so:

```json
{ "type_definitions": [
  { "type": "user", "relations": {} },
  { "type": "group",
    "relations": {
      "parent": { "directly_related_user_types": [{ "type": "group" }], "this": {} },
      "member": { "directly_related_user_types": [{ "type": "user" }, { "type": "group", "relation": "member" }], "this": {} },
    }
  },
] }
```

However, that is a breaking change due to how the relations are currently defined in the [protobuf files](https://github.com/openfga/api/blob/f10bb663ad633df8c6f7cdcfc7280b959ab47293/openfga/v1/authzmodel.proto#L50) as a map of usersets.

If we do introduce it, it should probably be bundled with other cleanup and breaking changes at a later point in time.

### Representing all objects

#### Drop support for `*`, introduce support for an alternative syntax, require specifying the type for which wildcards are valid

- To represent everyone in the subject field of a relationship tuple: `user=${type}:*` (note: it must include the type)
- To add everyone to the type restrictions of a certain relation, add `{ "type": "employee", "wildcard": {} }` (other alternatives considered were: `{ "type": "employee", "wildcard": true }`)

benefits: ability to properly represent everyone of a particular type
cons: breaking changes, `*` is not supported, and `employee:*` will be interpreted differently

#### Drop support for `*`, introduce support for an alternative syntax, introduce `allowPublic` on a type metadata 

- To represent everyone in the subject field of a relationship tuple: `user=${type}:*` (note: it must include the type)
- `allowPublic` on a type metadata means all type restrictions with no relations are now accepted

#### Keep support for `*`, interpret it differently

- `*` is kept, but will be interpreted as all the types that are allowed in type restrictions

benefits: no breaking changes, possible to seemlessly upgrade from a 1.0 to a 1.1 model
cons: inability to properly represent everyone of a particular type


We chose the first option in order to give developers the choice to better represent their models, and be more explicit in what they want to allow.

# Prior Art

Other projects that have a Zanzibar-like API have adopted similar patterns:

- SpiceDB - [Representing users as subjects](https://docs.authzed.com/guides/defining-a-subject-type#representing-users-as-subjects)
- Permify - [Modeling](https://docs.permify.co/docs/getting-started/modeling)
- Ory Keto - [Type declaration](https://www.ory.sh/docs/keto/reference/ory-permission-language#type-declaration)

## Unresolved Questions
