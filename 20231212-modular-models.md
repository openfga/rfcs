# Modular Models

## Meta

- **Name**: Modular Models
- **Start Date**: 2023-12-12
- **Author(s)**: [aaguiarz](https://github.com/aaguiarz)
- **Status**: Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- **PR Link**: https://github.com/openfga/rfcs/pull/14/files
- **Relevant Issues**:
- **Supersedes**: N/A

## Summary

The FGA model currently needs to be defined in a single file. This RFC proposes a way to define the model in multiple files that can be combined into a single model when writing it to a FGA store.

## Definitions

- [OpenFGA DSL](https://openfga.dev/docs/configuration-language)

## Motivation

### Enable model composition and collaboration across multiple teams

Authorization logic is application/service specific. In companies with multiple teams building multiple components, each team should own the authorization logic for their domain.

Having all teams to maintain a model in a single file has several drawbacks:

  - Requires a more comprehensive PR process where all teams need to be involved.
  - The model can become large and difficult to understand.
  - It's likely that the authorization model can't be part of the same repository as the code that uses it.

### Enable different applications to perform different actions in different modules

In addition to enable different developers maintain their own modules, it should be possible to enable different applications to perform different actions in different modules. 

For example, applications belonging to the domain which owns a module should be able to write/delete tuples of the object types the module defines, but other applications outside of that domain should only be able to query relationships of those object types.

This functionality is not part of this RFC, but it could be built on top of it.

## Use Cases

To illustrate the use cases, we’ll use Atlassian as an example. Atlassian has several products. A couple of well known ones are Confluence (a wiki) and Jira (project management).

If Atlassian implemented authorization using OpenFGA they would have a set of core types that every application needs to be aware of, like:

```
type user
type organization
   define member : [user]
   define admin : [user]
type group
    define member : [user]
```    

Then, the `Confluence` team could define a model for their own resources:

```
type space
  relations
    define organization : [organization] 
type page
  relations
    define space : [space]
    define owner : [user]
```

And the `Jira` team would define their own model too:

```
type project
  relations
    define organization : [organization] 
type ticket
  relations
    define project : [project]
    define owner : [user]
```

To enable this scenario, we'd need to:

- Define a modules with a set of types that reflect the resources/permissions for an app/service.
- Be able to reference types in other modules.
- Be able to add relationships to types in other modules. This is something that might not be needed and will be discussed later.

## New DSL Constructs Needed

To enable the use cases defined above, we need:

- A way to name a module
- A way to import a module 
- A way to reference a type/relation from another model
- A way to add relations to types from another model

## Proposed Syntax

Below there's an example of how the models would look like with the proposed syntax.

`core.fga`
```
module core

type user
type organization
   define member : [user]
   define admin : [user]
type group
    define member : [user]
```

`confluence.fga`
```
module confluence
import "core.fga"

extend type core.organization
  relations
    define can_create_space : [core.user] or admin
    
type space
  relations
    define organization : [core.organization] 
    
type page
  relations
    define space : [space]
    define owner : [core.user]
```

`jira.fga`
```
module jira
import "core.fga"

extend type core.organization
  relations
    define can_create_project : [core.user] or admin

type project
  relations
    define organization : [core.organization] 
    
type ticket
  relations
    define project : [project]
    define owner : [core.user]
```

## Proposed Implementation

The simplest way to implement this is to do it at the language + CLI level. 

For example, with the models above, the CLI will combine the files and generate a model like the one below:

`model.fga`
```
type core.user
type core.organization
   define member : [core.user]
   define admin : [core.user]
   define can_create_space : [core.user] or admin
   define can_create_project : [core.user] or admin

type core.group
    define member : [core.user]
    
type confluence.space
  relations
    define organization : [core.organization] 
    
type confluence.page
  relations
    define space : [confluence.space]
    define owner : [core.user]

type jira.project
  relations
    define organization : [core.organization] 
    
type jira.ticket
  relations
    define project : [jira.project]
    define owner : [core.user]
```

This implies check/write operations need to use the fully qualified type names. 

For example, to check if the user `anne` user is the owner of the `engineering` space, the check would be:

`check` (
  `user`: `"core.user:anne"`, 
  `relation`: `"owner"`, 
  `object` : `"confluence.space:engineering"`)

## Handling default modules

The DSL will still support defining types without a module, e.g.

```
type user
type organization
   define member : [user]
   define admin : [user]
```

If you want to reference these types from another module, you can use the `default` module name, which would be a protected keyword that can't be used to name your own modules.

```
type space
  relations
    define organization : [default.organization] 
```

This enables introducing modules in scenarios where you already have a production model that you can't easily change without impacting existing applications, as if you add existing types to a module, the types will be renamed to include the module name.

## Open Questions

- Should we allow adding relations to types from other modules? 

It's common that for each resource type, you need to add a relation in a higher level to allow creating instances of that resource. For example, to know if a user can create a Project in Jira, you'd have a `can_create_project` permission in the `organization` type.

In the examples above, we exemplified a way where the relation can be added in the Jira module using the `extend type base.organization` syntax.

The alternative is to have the relation added to the `core` module directly. However, if we want to optimize for having each application team operate independently, enabling each team to add relations.

- Is it feasible to implement Modules at the CLI level without any modification to the JSON format and the server? This implies the server won't know what a Module is, and can be problematic if we want to add permission-per-module in the future. We'll need to infer module names from the type names.

- Should we bump the schema version to 1.2 if we introduce this change? It's not a breaking change, so we don't **need** to version it. 

