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

## CLI Flags for Flexible Testing

Add new flags to the `kyverno test` command for different testing modes:

```bash
# Test with fake/mocked data (current behavior)
kyverno test .

# Test with real lookups for non-resource CEL libraries
kyverno test . --lookup

# Test with real cluster resource lookups + other real lookups
kyverno test . --lookup --cluster
```

## Behavior Changes

**Without `--lookup` flag (current behavior):**
- All CEL library functions return mocked/static data
- Resource lookups use context files or fake clients

**With `--lookup` flag (new behavior):**
- **Resource Library**: Uses context files with fake client by default
  - Add `--cluster` flag to use real Kubernetes API instead
- **HTTP Library**: `http.Get()`, `http.Post()` make real HTTP(S) requests to external endpoints
- **User Library**: `parseServiceAccount()` and other user functions work with actual user context
- **ImageData Library**: `image.GetMetadata()` fetches real metadata from OCI registries
- **GlobalContext Library**: `globalContext.Get()` retrieves real global context data

### Resource Library Options

**Option 1: Isolated Testing (default with --lookup)**
- Load resource manifests from context files into fake Kubernetes client
- Enables testing with controlled, predictable resource data
- No cluster connection required
- Example: `kyverno test . --lookup`

**Option 2: Real Cluster Testing (--lookup --cluster)**
- Connect to actual Kubernetes API server
- Perform real `resource.Get()`, `resource.List()`, `resource.Post()` calls
- Requires cluster access and credentials
- Example: `kyverno test . --lookup --cluster`

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

- **Dependency on External Systems**: HTTP, image, and cluster lookups become dependent on network connectivity and external service availability
- **Performance Impact**: Real network calls are slower than fake context lookups, increasing test execution time
- **Cost Implications**: External API calls could incur costs or rate limiting
- **Flakiness**: Network issues or external service outages could cause intermittent test failures
- **Authentication Complexity**: Real cluster access requires proper kubeconfig and credentials
- **Test Isolation**: Real cluster testing (--lookup --cluster) may interfere with other cluster resources

# Alternatives

# Prior Art

# Unresolved Questions


# CRD Changes (OPTIONAL)