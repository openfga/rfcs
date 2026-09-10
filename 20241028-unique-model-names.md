# Unique Model Names

## Meta

- **Name**: Unique Model Names
- **Start Date**: 2024-10-25
- **Last Updated Date**: 2024-10-25
- **Author(s)**: [aaguiarz](https://github.com/aaguiarz)
- **Status**: Draft 
- **PR Link**:
- **Relevant Issues**:
- **Supersedes**: N/A

## Summary

When [creating a model](https://openfga.dev/api/service#/Authorization%20Models/WriteAuthorizationModel), OpenFGA does not allow to provide a name,and it will return a unique id.

This RFC proposes a way to specify a unique name when writing a model.

## Motivation

In some cases, developers would benefit from having an external identifier for the model. 
  
For example, when deploying applications in development/staging environments a new model needs to be created in each deployment, and it desirable to have a predictable identifier for the model. Given OpenFGA creates a different Model ID each time, it's not possible. It needs to be stored in a secret storage vault, and retrieved at runtime, which adds friction. Some OpenFGA developers, for example, keep a database table that has a Github commit hash and the equivalent Model ID.

## Requirements

  - It should be possible to upgrade to OpenFGA version that implements this feature without downtime.
  - The OpenFGA [ReadAuthorizationModels endpoint](https://openfga.dev/api/service#/Authorization%20Models/ReadAuthorizationModels) endpoint should support filtering by name. 
  - The model name should be unique per store, not unique per OpenFGA instance.

## Proposed Solution

  - Add a `name` parameter to the (https://openfga.dev/api/service#/Authorization%20Models/WriteAuthorizationModel). 
  - Validate that the name is unique. If a database constraint is used, a migration should be created that sets the Model Name = Model ID.
  - Add a `name` parameter to the [ReadAuthorizationModels endpoint](https://openfga.dev/api/service#/Authorization%20Models/ReadAuthorizationModels).