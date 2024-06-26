# Modular Models

## Meta

- **Name**: Modular Models
- **Start Date**: 2023-12-12
- **Last Updated Date**: 2024-02-18
- **Author(s)**: [aaguiarz](https://github.com/aaguiarz)
- **Status**: Approved 
- **PR Link**: https://github.com/openfga/rfcs/pull/14/files
- **Relevant Issues**:
- **Supersedes**: N/A

## Summary

The FGA model currently needs to be defined in a single file. This RFC proposes a way to define the model in multiple files called modules that can be combined into a single model when writing it to a FGA store.

## Definitions

- [OpenFGA DSL](https://openfga.dev/docs/configuration-language)

## Motivation

### Enable model composition and collaboration across multiple teams

Authorization logic is application/service specific. In companies with multiple teams building multiple components, each team should own the authorization logic for their domain.

Having all teams to maintain a model in a single file has several drawbacks:

  - Requires a more comprehensive PR process where all teams need to be involved.
  - The model can become large and difficult to understand.

### Enable different applications to perform different actions in different modules

In addition to enable different developers maintain their own modules, it should be possible to enable different applications to perform different actions in different modules. 

For example, applications belonging to the domain which owns a module should be able to write/delete tuples of the object types the module defines, but other applications outside of that domain should only be able to query relationships of those object types.

This functionality is not part of this RFC, but the solution we implement should enable this functionality in the future.

## Use Cases

To illustrate the use cases, we’ll use Atlassian as an example. Atlassian has several products, including Confluence (a wiki) and Jira (a project management tool).

If Atlassian implemented authorization using OpenFGA they would have a set of core types that every application needs to be aware of, like:

```
type user
type organization
  relations
   define member : [user]
   define admin : [user]
type group
  relations
    define member : [user]
```    

Then, the `Confluence` team could define a separate module for their own resources:

```
type space
  relations
    define organization : [organization] 
type page
  relations
    define space : [space]
    define owner : [user]
```

And the `Jira` team would define a separate module for their resources:

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

- Define modules with a set of types that reflect the resources/permissions for an app/service.
- Be able to reference types in other modules.

Additionally, we'd need to be able to extend types from other modules to add relations to them. For example, the `Confluence` team would need to add a `can_create_space` relation to the `organization` type to allow users to create spaces.

## New DSL Constructs Needed

To enable the use cases defined above, we need:

- A way to name a module.
- A way to reference a type/relation from another module.
- A way to add relations to types from another module.
- A way to compile multiple modules into a single model.

## Proposed Solution

When using Modular Models, developers will need to create a `fga.mod` file that will list all modules that are part of the model, for example:

```yaml
# fga.mod
schema: 1.2
contents:
  - core.module.fga
  - jira.module.fga
  - confluence.module.fga
```
> **Note**: Adding `module` in the file name can help create a more readable naming convention by describing what content is in the file, but it is not required to compose modular models. `jira.module.fga`, `jira-module.fga`, and `jira.fga` will all work as long as the `.fga` file extension is used.

Each module will define its own types, and have the ability to extend types in other modules:

`core.module.fga`
```
module core

type user
type organization
   relations
      define member : [user]
      define admin : [user]
type group
   relations
      define member : [user]
```

`confluence.module.fga`
```
module confluence

extend type organization
  relations
    define can_create_space : user or admin
    
type space
  relations
    define organization : organization
    
type page
  relations
    define space : [space]
    define owner : [user]
```

`jira.module.fga`
```
module jira

extend type organization
  relations
    define can_create_project : user or admin

type project
  relations
    define organization : [organization] 
    
type ticket
  relations
    define project : [project]
    define owner : [user]
```

The `fga.mod` file will be used to generate a single model file by the CLI that can be used by the FGA server.

```shell
fga model write --file ./fga.mod
```

Below is the model that would be stored in the FGA store:

```python
model
  schema 1.2

type user

type organization
  relations
    define member : [user]
    define admin : [user]
    define can_create_space :  user or admin
    define can_create_project : user or admin

type group
  relations
    define member : [user]

type project
  relations
    define organization : [organization] 
    
type ticket
  relations
    define project : [project]
    define owner : [user]

type space
  relations
    define organization : organization
    
type page
  relations
    define space : [space]
    define owner : [user]
```

Module information will be stored in the JSON representation of the model as metadata. Using that information it should be possible to reconstruct each module, and use module information for authorization decision in the future (e.g. certain applications can only write tuples to certain modules).

This metadata will be stored in the below structure, and will be stored as part of the existing `metadata` for types and relations, and a new `metadata` property will be added for conditions. File and module metadata will only be added to relations that originate from extended types and not to relations that come from the original type.

The module name will be stored at the top level of the `metadata` object as we envisage that it will be important for future features such as FGA-on-FGA and will therefore become a native piece of the authorization model. The file will be stored under a new `source_info` object that may grow to include other source specific properties.

```json
{
  ...,
  "module": string,
  "source_info": {
    "file": string
  }
}
```

For example the `organization` type from `core.module.fga` would look like below (fields other than `metadata` removed for brevity):

```json
{
  "type": "organization",
  "relations": { ... },
  "metadata": {
    "relations": {
      "member": { ... },
      "admin": { ... },
      "can_create_space": {
        "module": "confluence",
        "source_info": {
          "file": "confluence.module.fga"
        }
      },
      "can_create_project": {
        "module": "jira",
        "source_info": {
          "file": "jira.module.fga"
        }
      }
    },
    "module": "core",
    "source_info": {
      "file": "core.module.fga"
    }
  }
}
```

## Managing Modules in Github

To enable collaboration across multiple teams, the `.github/CODEOWNERS` file in the repository can be used to define which team owns which module. Members of that team will need to approve PRs that change the module.

```
fga.mod @atlassian/core
core.module.fga @atlassian/core
jira.model.fga @atlassian/jira
confluence.model.fga @atlassian/confluence
```

## High Level Implementation Details

- The Language will be upgraded to support the `module` and `extends` keywords.
- The CLI will be upgraded to support the new `fga.mod` 
- The VS Code Extension will be upgraded to support the new `fga.mod` files and the new language constructs.
- The JSON model format will be upgraded to include module information as metadata for types and relations.

The OpenFGA server does not need to be changed, beyond supporting the new JSON format and allowing schema version `1.2`. SDKs do not need to be changed beyond being refreshed to allow for the new Authorization Model type with the additional fields.

### Type Extension

With one of the main drivers behind modular models being to help reduce friction between teams working on a singular model it was decided that type extension should be included to allow extending types without the need for the owning team to take ownership of these relations. In the future, it would be possible to allow restricting the usage of these relations to within the module they are extended within.

In order to support type extension there must be some restrictions on the usage to avoid any clashes or unexpected behavior. All the below examples are not allowed:

* An extended type must exist
```python
module confluence

extend type non-existent
  relations
    define can_create_project : user or admin
```
* A type can only be extended once per file
```python
module confluence

extend type organization
  relations
    define can_create_project : user or admin

extend type organization
  relations
    define can_delete_project : admin
```
* If the `extend` keyword is used then there must be relations declared

```python
module confluence

extend type organization
```
* There cannot be any relations that would result in duplicates once the modular model has been compiled
```python
# confluence.module.fga
module confluence

extend type organization
  relations
    define can_create_project : user or admin
```
```python
# jira.module.fga
module jira

extend type organization
  relations
    define can_create_project : user or admin
```

### Conditions

Similar to types, conditions defined in any module can be used throughout the modular model. In the future, it would be possible to restrict the usage of these conditions to within the module they are extended within.

In order to support this usage there are some restrictions in place. All the below examples are not allowed:

* Conditions must be unique across the module project
```python
# confluence.module.fga
module confluence

condition non_expired_grant(current_time: timestamp, grant_time: timestamp, grant_duration: duration) {
  current_time < grant_time + grant_duration
}
```
```python
# jira.module.fga
module jira

condition non_expired_grant(current_time: timestamp, grant_time: timestamp, grant_duration: duration) {
  current_time < grant_time + grant_duration
}
```

## Other options evaluated

### Prefixing types with the module name

We considered prefixing type names with module names (e.g. "confluence.space"). We decided against because:

- Refactoring a model in different modules would be a very impactful change. Changing type names implies changing application code and tuples.

- Existing types are not namespaced, which implies that every developer adopting modules will need to change their application and tuples.

If developers want to namespace their types, they can do it without changes by using namespaced type names (e.g. `confluence.space`).

### Importing Modules

Instead of having a `fga.mod` file, we could have a way to import other modules, and use the CLI to specify all modules needed to create a model, e.g.

`fga model write --file ./core.module.fga --file ./jira.module.fga --file ./confluence.module.fga`

We decided to not to because:

  - It would make VS Code tooling more complex. To find a type definition for validation/coloring we'd need to incrementally traverse all `import` clauses. 

  - Having a `fga.mod` helps clearly identify which modules are part of the model instead of relying on a CLI command executed at build time.

### Schema Version

We decided to bump the schema to 1.2 to signal that the model was built using the new modular model feature. 

The Playground not will not be updated immediately to support modular models, and it will use the schema version to determine if it can display the model.
