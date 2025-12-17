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

This feature enables real lookups for all Kyverno CEL libraries in CLI tests, allowing policies to interact with actual external systems instead of mocked data. Currently, the CLI test command uses fake context providers that return static data, preventing comprehensive testing of policies that use CEL libraries for dynamic lookups against Kubernetes APIs, HTTP endpoints, registries, and other external services. This enhancement introduces a `--lookup` flag that enables real interactions across all CEL libraries during testing.

# Definitions
[definitions]: #definitions

- **CLI Test Command**: The `kyverno test` command that validates policies against test cases without requiring a running cluster
- **CEL Libraries**: Common Expression Language (CEL) extensions that provide [additional functions](https://kyverno.io/docs/policy-types/cel-libraries/) for Kyverno policies, including resource, HTTP, user, image, imageData, and globalContext libraries
- **Fake Context Provider**: A testing implementation that provides static, predefined data instead of making real external calls
- **Real Lookup**: Dynamic data retrieval from actual external systems (Kubernetes API, HTTP endpoints, registries, etc.) during policy evaluation

# Motivation
[motivation]: #motivation

- **Why should we do this?** The CLI test command currently uses fake context providers that return static, mocked data for all CEL library operations. This prevents comprehensive testing of policies that depend on real external systems, creating a significant gap between test and production environments where policies may behave differently with actual data sources.

- **What use cases does it support?**
  - Testing policies that query Kubernetes APIs via the [resource library](https://kyverno.io/docs/policy-types/cel-libraries/#resource-library)
  - Validating HTTP(S) endpoint responses using the [HTTP library](https://kyverno.io/docs/policy-types/cel-libraries/#http-library)
  - Testing user authentication logic with the [user library](https://kyverno.io/docs/policy-types/cel-libraries/#user-library)
  - Validating image metadata from real registries using the [imageData library](https://kyverno.io/docs/policy-types/cel-libraries/#imagedata-library)
  - Testing policies with real global context data via the [globalContext library](https://kyverno.io/docs/policy-types/cel-libraries/#globalcontext-library)
  - CI/CD pipelines that need to validate complete policy behavior against actual infrastructure

- **What is the expected outcome?** Users will be able to run `kyverno test --lookup` to enable real interactions across all CEL libraries, providing confidence that policies work correctly with actual external systems and data sources.

# Proposal

This feature enables real lookups for all Kyverno CEL libraries by introducing a `--lookup` flag that switches from fake context providers to real external interactions during CLI testing.

## Proposed Solution

Add a new `--lookup` flag to the `kyverno test` command:

```bash
# Test with fake/mocked data (current behavior)
kyverno test .

# Test with real lookups for all CEL libraries
kyverno test . --lookup
```

### Behavior Changes

**Without `--lookup` flag (current behavior):**
- All CEL library functions return mocked/static data
- Resource lookups use context files or fake clients
- HTTP calls return predefined responses
- User library functions return mock user data
- Image operations use cached/stored metadata

**With `--lookup` flag (new behavior):**
- **Resource Library**: `resource.Get()`, `resource.List()`, `resource.Post()` make real Kubernetes API calls
- **HTTP Library**: `http.Get()`, `http.Post()` make real HTTP(S) requests to external endpoints
- **User Library**: `parseServiceAccount()` and other user functions work with actual user context
- **ImageData Library**: `image.GetMetadata()` fetches real metadata from OCI registries
- **GlobalContext Library**: `globalContext.Get()` retrieves real global context data

## Examples

### Test Configuration Example
```yaml
# kyverno-test.yaml (unchanged - works with both fake and real lookups)
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: policy-test
policies:
  - policy.yaml
resources:
  - pod.yaml
results:
  - policy: policy.yaml
    rule: validate-resource
    result: pass
```

### Policy Example Using CEL Libraries
```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: comprehensive-validation
spec:
  variables:
    - name: configMap
      expression: 'resource.Get("v1", "configmaps", object.metadata.namespace, "app-config")'
    - name: externalData
      expression: 'http.Get("https://api.example.com/validate")'
    - name: imageInfo
      expression: 'image.GetMetadata(object.spec.containers[0].image)'
  validations:
    - expression: 'variables.configMap.data.enabled == "true"'
    - expression: 'variables.externalData.status == "valid"'
    - expression: 'variables.imageInfo.config.os == "linux"'
```

When run with `kyverno test . --lookup`, the policy will:
- Make real `resource.Get()` calls to the Kubernetes API
- Perform actual `http.Get()` requests to external services
- Fetch real image metadata from OCI registries
- Use actual user context for authentication functions
- Access real global context data when available

## Key Design Decisions

1. **Opt-in Behavior**: Real HTTP lookups are disabled by default to maintain backward compatibility
2. **Isolated Resource Testing**: Resource lookups always use isolated testing with fake client for predictability
3. **Scoped Implementation**: Initially target resource and HTTP(S) libraries, with extensibility for future libraries
4. **Clear Separation**: Resource lookups use static data, HTTP lookups use real network calls

# Implementation

## Link to the Implementation PR


# Migration (OPTIONAL)

This feature is designed to be backward compatible with no breaking changes.

## Backward Compatibility

- **No API Changes**: Existing test files work unchanged
- **Opt-in Feature**: `--lookup` flag enables new behavior when desired
- **Safe Defaults**: Fake context providers remain the default behavior

# Drawbacks


# Alternatives

# Prior Art

# Unresolved Questions


# CRD Changes (OPTIONAL)