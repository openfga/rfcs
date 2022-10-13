# Add Type Restrictions to the DSL Syntax

## Meta

- **Name**: Add Type Restrictions to the DSL Syntax
- **Start Date**: 2022-10-12
- **Author(s)**: [craigpastro](https://github.com/craigpastro)
- **Status**: Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- **RFC Pull Request**: <https://github.com/openfga/rfcs/pull/8>
- **Relevant Issues**:
- **Supersedes**: N/A

## Summary

The DSL syntax is used to specify authorization models. Type restrictions on an authorization model will allow us to restrict the types of users that may be directly related to (type, relation) pairs. For example, we may restrict that the parent of a folder must be a folder itself. In this RFC we are proposing a syntax to add type restrictions to the DSL.

See the [Add Type Restrictions to the JSON Syntax RFC](./20220831-add-type-restrictions-to-json-syntax.md) for further background.

## Definitions

- [OpenFGA DSL](https://openfga.dev/docs/configuration-language)
- [ListObjects Endpoint](https://openfga.dev/api/service#/Relationship%20Queries/ListObjects)
- [What is a user?](https://openfga.dev/docs/concepts#what-is-a-user)
- Reverse Expansion: Normally [expand](https://openfga.dev/docs/interacting/relationship-queries#expand) takes an object and relation and returns the first leaves of users who are related. Reverse expand would be the opposite, taking a user and a relation and returning the objects which are related.

## Motivation

### Optimize ListObjects

The main reason for this change is to optimize the [ListObjects](https://github.com/openfga/rfcs/blob/main/20220714-listObjects-api.md) endpoint. ListObjects needs to traverse the permission graph in reverse, and by restricting the types of users that may be related to objects, we are able to reduce the number of edges we must traverse from each node.

### Improve the developer experience

Adding type restrictions may improve the understandablity of authorization models. One may see immediately that certain relations may only have certain types of users. For example, perhaps parents of folder can only be of type folder, or readers of documents can only be of type user.

## Type restrictions

### Types with no relations

It now becomes useful to allow types with no relations. For example, we may want to specify a `user` type which is the type for all concrete users. This will look like
```
type user
```

### Type restrictions

Let A be an authorization model with set of types T and set of relations R. A type restriction on A can be seen as a function from TxR to TxR*, where R* is R with the addition of an "empty" relation. That is, a type restriction takes any (type, relation) in A and takes it to the set of type-relation pairs, with possibly empty relation, that it may be directly related to.

#### Examples

1. Suppose A is an authorization model with a type folder and relation parent on folder. We may have a type restriction on A that takes (folder, parent) to the set {(folder, \_)}, meaning that parents of folders must be folders. (The "_" is used to indicate the empty relation.)
 2. Suppose A is an authorization model with type document and relation reader on document. We may have a type restriction on A that takes (document, reader) to the set {(user, \_), (group, member)}, meaning that the only users that may be readers of a document are users with type user, or a userset of group members.


### Syntax

Let A be an authorization model with set of types T and set of relations R, and recall that R* is the set R with the addition of an "empty" relation. We will write an element (t, r) of TxR* as t#r. The special case (t, _) is simply written as t. If the type restriction takes (t, r) to a set {(t1, r1), (t2, r2)}, then in our model we will write this as
```
type t
    relations
        define r: [t1#r1, t2#r2] as ...
```
If the type restriction takes (t, r) to the empty set we will simply write nothing in our model.

#### Example

Let's look at a concrete example. Consider the model:
```
type group
    relations
        define parent as self
        define member as self
        define guest as self
        define can_view as guest
```
Suppose we want the following restrictions:
1. parents of groups are groups
2. members of groups are employees or group members
3. guests of groups are users, employees, or group members

We can write the model with these type restriction as
```
type user

type employee

type group
    relations
        define parent: [group] as self
        define member: [employee, group#member] as self
        define guest: [user, employee, group#member] as self
        define can_view as guest
```

## Affect of type restrictions: writing tuples

Given a model with type restrictions, we can now only write tuples that respect the type restriction. For example, given our model above the following are valid tuples:
- object=group:X, relation=parent, user=group:Y
- object=group:X, relation=member, user=employee:susan
- object=group:X, relation=member, user=group:Y#member
- object=group:X, relation=guest, user=user:susan
- object=group:X, relation=guest, user=employee:mary
- object=group:X, relation=guest, user=group:Y#member

The following tuples would _not_ be valid:
- object=group:X, relation=parent, user=frankie       # group parents can only be groups
- object=group:X, relation=parent, user=user:frankie  # group parents can only be groups
- object=group:X, relation=guest, user=group:Y        # group guests can only be group members, and not just groups


## Further information

More information regarding the addition of type restriction may be found in [Add Type Restrictions to the JSON Syntax RFC](./20220831-add-type-restrictions-to-json-syntax.md).
