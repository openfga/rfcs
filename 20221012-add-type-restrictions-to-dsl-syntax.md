# Add Type Restrictions to the DSL Syntax

## Meta

- **Name**: Add Type Restrictions to the DSL Syntax
- **Start Date**: 2022-10-12
- **Author(s)**: [craigpastro](https://github.com/craigpastro)
- **Status**: Approved
- **RFC Pull Request**: <https://github.com/openfga/rfcs/pull/8>
- **Relevant Issues**:
- **Supersedes**: N/A

## Summary

The DSL syntax is used to specify authorization models. Type restrictions on an authorization model will allow us to restrict the types of users that may be directly related to (type, relation) pairs. For example, we may restrict that the parent of a folder must be a folder itself. In this RFC we are proposing a syntax to add type restrictions to the DSL, and updates related to this.

See the [Add Type Restrictions to the JSON Syntax RFC](./20220831-add-type-restrictions-to-json-syntax.md) for further background.

## Definitions

- [OpenFGA DSL](https://openfga.dev/docs/configuration-language)
- [ListObjects Endpoint](https://openfga.dev/api/service#/Relationship%20Queries/ListObjects)
- [What is a user?](https://openfga.dev/docs/concepts#what-is-a-user)
- Reverse Expansion: Normally [expand](https://openfga.dev/docs/interacting/relationship-queries#expand) takes an object and relation and returns the first leaves of users who are related. Reverse expand would be the opposite, taking a user and a relation and returning the objects which are related.

## Motivation

### Improve the developer experience

Adding type restrictions improves the understandability of authorization models. One may see immediately that certain relations may only have certain types of users. For example, parents of folders can only be of type folder, or readers of documents can only be of type user.

### Optimize ListObjects

Type restrictions will allow us to optimize the [ListObjects](https://github.com/openfga/rfcs/blob/main/20220714-listObjects-api.md) endpoint. ListObjects needs to traverse the permission graph in reverse, and by restricting the types of users that may be related to objects, we are able to reduce the number of edges we must traverse from each node.

## Schema version 1.1

Since this is a significant breaking change to the DSL we have decided to add a schema version to the DSL. The previous version of the DSL had schema version 1.0, and the schema version with type restrictions is 1.1. If you wish to use type restrictions in the DSL please add the following to the top of the model:

```
model
  schema 1.1
```

## Type restrictions

### Type restrictions

Let A be an authorization model with set of types T and set of relations R. A type restriction on A can be seen as a set-valued function from TxR to TxR*, where R* is R with the addition of an "empty" relation. That is, a type restriction takes any (type, relation) in A and takes it to the set of type-relation pairs (where the relation may be empty) that it may be directly related to. That is, if the pair (folder, viewer) has the type restriction (employee, \_), where "\_" is used to indicate the empty relation, then the only tuples of the form (object=folder:documents, relation=viewer, user=U) that are allowed are ones in which `U=employee:X`, where `X` is an employee id.

#### Examples

1. Suppose A is an authorization model with a type folder and relation parent on folder. We may have a type restriction on A that takes (folder, parent) to the set {(folder, \_)}, meaning that parents of folders must be folders.
 2. Suppose A is an authorization model with type document and relation reader on document. We may have a type restriction on A that takes (document, reader) to the set {(user, \_), (group, member)}, meaning that the only users that may be readers of a document are users with type user, or a userset of group members.

### Syntax

Let A be an authorization model with set of types T and set of relations R, and recall that R* is the set R with the addition of an "empty" relation. We will write an element (t, r) of TxR* as t#r. The special case (t, _) is simply written as t. If the type restriction takes (t, r) to a set {(t1, r1), (t2, r2)}, then in our model we will write this as
```
type t
  relations
    define r: [t1#r1, t2#r2] ...
```
If the type restriction takes (t, r) to the empty set we will simply write nothing in our model.

### Types with no relations

It now becomes useful to allow types with no relations. For example, suppose we want to define a `viewer` relation and retrict the types we may want to specify a `user` type which is the type for all concrete users. This will look like
```
type user
```

### The dropping of "as" and "self"

In the new DSL we will drop "as" and "self" keywords. We drop "as" by adding ":" to all relations, even ones with no type restrictions. That is, if previously we had `define editor as owner`, this becomes `define editor: owner`. With the addition of type restrictions means we may also drop `self` from the DSL. This is because if a pair (type, relation) has a non-empty set of type restrictions, then the assumption is that (type, relation) is a pair that allows a direct relation to type. Thus, previously if we defined a relation as
```
define viewer as self
```
we can now write this as
```
define viewer: [user]
```

The dropping of "self" will be extended to rewrite rules as well. If a rewrite contains `self` then it may be dropped as in the following examples:
- `define viewer as self or viewer from parent` -> `define viewer: [user] or viewer from parent`
- `define viewer as self and viewer from parent` -> `define viewer: [user] and viewer from parent`
- `define viewer as self but not banned` -> `define viewer: [user] but not banned`

### Wildcards

The syntax and semantics of wildcards have also been updated in the new DSL. Consider the (new) authorization model
```
type user

type document
  relations
    define viewer: [user]
```
Previously, writing a tuple with the wildcard symbol `document:X#viewer@*` meant that we intended the object `document:X` to be publically "viewable". In the new DSL, we will also add type restrictions to the wildcard symbol, and there will no longer be an untyped wildcard symbol. If you want to say that a document can be publically viewable for all `user`s, then you must add `user:*` to the type restrictions:
```
type user

type document
  relations
    define viewer: [user, user:*]
```
Then you can write tuples such as `document:X#viewer@user:*`, which means that `document:X` is viewable to all `user`s. If you had another type restriction, say `employee`, then you may instead write something like
```
type user

type employee

type document
  relations
    define viewer: [user, employee:*]
```
which means you could write tuples such as `document:Y#viewer@employee:*`, which means that `document:Y` is viewable to all `employee`s.

## Example

Consider the following authorization model:
```
type group
  relations
    define member as self

type document
  relations
    define viewer as self
```
Suppose we want the following restrictions:
1. viewer of documents can be users, groups, or members of a group
1. some documents may be viewable for all users
1. group members are users

We can write the model with these type restriction as
```
model
  schema 1.1

type user

type group
  relations
    define member: [user]

type document
  relations
    define viewer: [user, group, group#member, user:*]
```

## Effect of type restrictions: writing tuples

Given a model with type restrictions, we can now only write tuples that respect the type restriction. For example, given our model above the following are valid tuples:
- object=group:eng, relation=member, user=user:alice
- object=document:w, relation=viewer, user=user:beatrix
- object=document:x, relation=viewer, user=group:eng
- object=document:y, relation=viewer, user=group:hr#member
- object=document:z, relation=viewer, user=user:*

The following tuples would _not_ be valid:
- object=group:eng, relation=member, user=charlie           # group members must be related to type user
- object=group:eng, relation=member, user=group:iam         # group members must be related to type user
- object=group:eng, relation=member, user=group:iam#member  # group members must be related to type user
- object=document:x, relation=viewer, user=employee:diane   # employee is not a type of (document, viewer)
- object=document:y, relation=viewer, user=*                # untyped * are no longer allowed


## Further information

More information regarding the addition of type restriction may be found in [Add Type Restrictions to the JSON Syntax RFC](./20220831-add-type-restrictions-to-json-syntax.md).
