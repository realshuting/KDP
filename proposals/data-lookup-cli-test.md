# Meta
[meta]: #meta
- Name: Data Lookup Support for CLI Test Command
- Start Date: 2025-12-16
- Author(s): @realshuting

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)

# Overview
[overview]: #overview

This feature implements real lookup capabilities for the Kyverno CLI test command, enabling comprehensive testing of policy definitions that use CEL libraries for resource and HTTP(S) lookups. Currently, the CLI test command only supports static fake data through values.yaml files, limiting the ability to thoroughly test policies that perform dynamic lookups against Kubernetes API server or external HTTP(S) endpoints. This enhancement will allow users to test policy behavior against real cluster state and external services, ensuring policies work correctly in production environments.

# Definitions
[definitions]: #definitions

- **CLI Test Command**: The `kyverno test` command that validates policies against test cases without requiring a running cluster
- **Values.yaml**: Configuration files containing static test data that is currently used to mock external dependencies
- **CEL Libraries**: Common Expression Language (CEL) extensions that provide [additional functions](https://kyverno.io/docs/policy-types/cel-libraries/) for Kyverno policies, including resource lookups and HTTP(S) requests
- **Real Lookup**: Dynamic data retrieval from actual sources (Kubernetes API server, external HTTP(S) endpoints) during policy evaluation

# Motivation
[motivation]: #motivation

- **Why should we do this?** Currently, the CLI test command uses a fake context provider that only supports static data from values.yaml files. This severely limits the ability to test policies that use CEL libraries for dynamic lookups, as users cannot validate that URLs are correct, API calls work properly, or that policies behave correctly with real cluster state. This creates a gap between testing and production environments.

- **What use cases does it support?**
  - Testing policies that validate resource existence or properties via [resource](https://kyverno.io/docs/policy-types/cel-libraries/#resource-library) library calls
  - Validating HTTP(S) endpoint availability and response handling in policies using [http](https://kyverno.io/docs/policy-types/cel-libraries/#http-library) library
  - Ensuring policies work correctly with real cluster state rather than mocked data
  - Comprehensive testing of policy logic that depends on external data sources
  - CI/CD pipelines that need to validate policy behavior against actual infrastructure

- **What is the expected outcome?** Users will be able to run `kyverno test` commands that perform real API calls and HTTP(S) requests, providing confidence that policies will work correctly in production. This enables thorough testing of policy definitions before deployment, reducing the risk of policy failures in live environments.

# Proposal

This feature introduces a new CLI flag `--lookup` to the `kyverno test` command that enables real HTTP(S) requests for HTTP lookups while providing isolated testing for resource lookups. The test command will:

1. **For Resource Lookups**: Load resource manifests from `contextResources` directories/files into a Kubernetes fake client for controlled testing
2. **For HTTP(S) Lookups**: Make real HTTP(S) requests to external endpoints for HTTP operations
3. **Maintain Backward Compatibility**: When the flag is not provided, behavior remains unchanged (fake context)

## Examples

### Proposed Behavior (Real Lookups)
```bash
# Test with real HTTP(S) and Kubernetes resource lookups
kyverno test . --lookup
```

### Policy Example Using Real Lookups
```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: restrict-image-registries
spec:
  validationActions:
    - Deny
  evaluation:
    background:
      enabled: false
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  variables:
    - name: allContainers
      expression: >-
        object.spec.containers 
        + object.spec.?initContainers.orValue([]) 
        + object.spec.?ephemeralContainers.orValue([])
    - name: cm
      expression: >-
        resource.Get("v1", "configmaps", "kube-system", "allowed-registry")
    - name: allowedRegistry
      expression: "variables.cm.data[?'registry'].orValue('')"
  validations:
    - expression: "variables.allContainers.all(c, c.image.startsWith(variables.allowedRegistry))"
      messageExpression: '"image must be from registry: " + string(variables.allowedRegistry)'
```

### Test Configuration for Resource Lookups
```yaml
# kyverno-test.yaml - Isolated testing with context resources
apiVersion: cli.kyverno.io/v1alpha2
kind: Test
metadata:
  name: test-resource-lookup-isolated
policies:
  - policy.yaml
resources:
  - pod.yaml
contextResources:  # Directories/files containing resources for lookups
  - mocks/configmaps.yaml  # Specific file with resources
results:
  - policy: policy.yaml
    rule: check-configmap-exists
    result: pass
```

When run with `kyverno test . --lookup`, this will:
- Load all resources from `mocks/configmaps.yaml` into a fake client for isolated resource testing
- Execute policy lookups against these static resources
- Enable real HTTP(S) lookups for any HTTP operations in the policy

## Key Design Decisions

1. **Opt-in Behavior**: Real HTTP lookups are disabled by default to maintain backward compatibility
2. **Isolated Resource Testing**: Resource lookups always use isolated testing with fake client for predictability
3. **Scoped Implementation**: Initially target resource and HTTP(S) libraries, with extensibility for future libraries
4. **Clear Separation**: Resource lookups use static data, HTTP lookups use real network calls

# Implementation

## Schema Changes

Add the `contextResources` field to the `Test` struct:

```go
type Test struct {
    // ... existing fields ...

    // ContextResources specifies directories/files containing resource manifests
    // to be loaded into the fake client for isolated testing of resource lookups
    ContextResources []string `json:"contextResources,omitempty"`

    // ... existing fields ...
}
```

## Implementation Logic

When `--lookup` flag is used:
- Load resources from `contextResources` paths into fake Kubernetes client
- Enable real HTTP(S) client for HTTP library operations
- All resource lookups use the fake client with loaded resources
- All HTTP lookups use real network calls

## Link to the Implementation PR


# Migration (OPTIONAL)

This feature is designed to be backward compatible with no breaking changes.

# Drawbacks


# Alternatives

# Prior Art

# Unresolved Questions


# CRD Changes (OPTIONAL)

This KDP entails changes to the CLI test CRD to support resource lookups:

- **Version Bump**: Bump API version from `cli.kyverno.io/v1alpha1` to `cli.kyverno.io/v1alpha2` due to addition of new field
- **Test**: Add new `contextResources` field (`[]string`) to specify directories/files containing resource manifests for isolated testing
- **Backward Compatibility**: The new field is optional, existing v1alpha1 test configurations remain valid
