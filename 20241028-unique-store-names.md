# Unique Store Names

## Meta

- **Name**: Unique Store Names
- **Start Date**: 2024-10-25
- **Last Updated Date**: 2024-10-25
- **Author(s)**: [aaguiarz](https://github.com/aaguiarz)
- **Status**: Draft 
- **PR Link**:
- **Relevant Issues**:
- **Supersedes**: N/A

## Summary

When [creating a store](https://openfga.dev/api/service#/Stores/CreateStore), OpenFGA allows providing a store name, and it will return a unique id.

This RFC proposes a way to configure OpenFGA in a way that the store name can be unique.

## Motivation

In some cases, developers would benefit from having an external identifier for the store. Some examples are:  

  - The application is architected to use one store per tenant, and they need to map the internal tenant ID to the store ID.
  
  - When deploying applications in development/staging environments a new store needs to be created in each deploy, and it desirable to have a predictable identifier for the store. Given OpenFGA creates a different Store ID each time, it's not possible. It needs to be stored in a secret storage vault, and retrieved at runtime, which adds friction

## Requirements

  - Existing OpenFGA deployments that have duplicated names should still work.
  - OpenFGA [GetStores endpoint](https://openfga.dev/api/service#/Stores/GetStore)  endpoint should support filtering by name. Given it's possible that there could be more than one store with the same name, it needs to return an array. If the store name is unique, it will return an array with a single element.

## Proposed Solution

  - Add a configuration option to OpenFGA to enable unique store names.
  - Add a `name` parameter to the [GetStores endpoint](https://openfga.dev/api/service#/Stores/GetStore) that returns an array of stores.
  - Modify the storage adapters to validate that store names are unique when creating them. Given it is required to also support duplicated store names, we can't rely on database constraints.

