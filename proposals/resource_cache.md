# Meta
[meta]: #meta
- Name: Resource Cache
- Start Date: Aug 7, 2023
- Update Date: Sep 24th, 2023
- Author(s): @JimBugwadia

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
  - [Kubernetes resource](#kubernetes-resource)
  - [External API Call](#external-api-call)
  - [Other requirements](#other-requirements)
    - [Metrics](#metrics)
    - [Failure handling](#failure-handling)
    - [Failure](#failure)
  - [Link to the Implementation PR](#link-to-the-implementation-pr)
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Adding resource cache entry to a policy](#adding-resource-cache-entry-to-a-policy)
    - [Implementation](#implementation-1)
      - [Limited to known types](#limited-to-known-types)
      - [Support for any resource type](#support-for-any-resource-type)
    - [Drawbacks](#drawbacks-1)
  - [Add caching to API calls](#add-caching-to-api-calls)
- [Prior Art](#prior-art)
- [Limitations](#limitations)
- [Open Items](#open-items)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)

# Overview
[overview]: #overview

Optional caching of any Kubernetes resource.

# Motivation
[motivation]: #motivation

From: https://github.com/kyverno/kyverno/issues/4459

> In some cases to properly validate a resource we need to examine other resources. Particularly for Ingress and Istio Gateways/VirtualServices, we need to look at all the other Ingress/virtualservices or services in the cluster. At large scale, we are finding that Kyverno struggles to handle repeatedly pulling thousands of resources using the context apiCall variables. On a cluster with around 4,000 Service objects and 3,000 Ingresses, we found that a policy validating Istio VirtualService destinations against Services (ensuring the target exists) was taking > 10s for all measurements (p50, p95 and p99), and the webhook timeout was exceeded. Another policy that validates an Ingress doesn't duplicate a hostname from another Ingress had p95 execution times over 5 seconds. During this time the controllers were at/below requested values for CPU/memory and otherwise had no other indicator of performance problems.

# Proposal

There are two parts to this feature:
1. Allow users to manage which resources should be cached
2. Allow policy rules to reference cached resource data

Users can manage which resources to cache by creating a new custom resource called `ContextEntry` provided by Kyverno. This will decouple the creation and usage of a cache entry. 

A `ContextEntry` will be of two types:
1. `Resource`: A resource is a Kubernetes resource that should be cached, to create a `ContextEntry` of resource type, the following resource should be created:

```yaml
apiVersion: kyverno.io/v2alpha1
kind: ContextEntry
metadata:
  name: ingress
spec:
  resource:
    group: "apis/networking.k8s.io"
    version: "v1"
    kind: "ingresses"
    namespace: apps
```

This resource allows authors to declare what Kubernetes resource should be cached. The `group` and `version` are optional. If not specified, the preferred versions should be used. An optional `namespace` can be used to only cache resources in the namespace, rather than across all namespaces which is the behavior is a namespace is not specified.

2. `External`: An external is an external API call response that should be cached, to create a `ContextEntry` of the external type, The following resource should be created:

```yaml
apiVersion: kyverno.io/v2alpha1
kind: ContextEntry
metadata:
  name: ingress
spec:
  external:
    url: https://svc.kyverno/example
    caBundle: |-
      -----BEGIN CERTIFICATE-----
      -----REDACTED-----
      -----END CERTIFICATE-----
    refreshIntervalSeconds: 10
```

This allows authors to declare what API response should be cached. The `url` is the URL where the request will be sent. `caBundle` is a PEM-encoded CA bundle that will be used to validate the server certificate. The `refreshIntervalSeconds` is the interval after which the URL will be reached again to refresh the entry.

To reference these cache entries in a policy, we can add them to the context variable as follows,
```yaml
context:
  - name: ingresses
    contextEntry:
      name: ingress
      jmespath: "ingresses | items[].spec.rules[].host"
```

`contextEntry.name` is the name of the entry to be used. The JMESPath is optional and is applied to add a resulting subset of the resource data to the rule context.

# Implementation

Resource cache will require an in-memory cache that will be stored in every controller. We will store both types of context entries as follows:

## Kubernetes resource

Kyverno will use Kubernetes dynamic client to create generic informers and listers to cache any Kubernetes resource. These listers will then be stored in the cache and will be accessed when they are referenced in a policy.

## External API Call

Kyverno will store these in the same cache, External API cache entry will require polling to update the cache entry at the specified interval.

## Other requirements

### Metrics

It would be useful to add cache metrics for observability and troubleshooting.

### Failure handling

The assumption is that the cache is kept up-to-date. We will need to think about any potential race conditions, especially during startup, and how to handle scenarios where the cache is not populated.

### Failure 

## Link to the Implementation PR

TBD

# Migration (OPTIONAL)

There is no automated migration. 

However, to leverage resource caching, users can convert API calls to the new `resourceCache` declaration.

Here are some API calls from sample policies along with the corresponding `resourceCache` declarations:

https://kyverno.io/policies/other/e-l/ensure-production-matches-staging/ensure-production-matches-staging/

```yaml
    context:
    - name: deployment_count
      apiCall:
        urlPath: "/apis/apps/v1/namespaces/staging/deployments"
        jmesPath: "items[?metadata.name=='{{ request.object.metadata.name }}'] || `[]` | length(@)"
```

can be converted to:

```yaml
    context:
    - name: deployment_count
      resourceCache:
        group: "apps"
        version: "v1"
        kind: "deployments"
        namespace: "staging"
        jmesPath: "items[?metadata.name=='{{ request.object.metadata.name }}'] || `[]` | length(@)"
```

# Drawbacks

# Alternatives

## Adding resource cache entry to a policy

Users can manage which resources to cache using the same mechanism that is currently used for ConfigMap resources i.e. adding a label `cache.kyverno.io/enabled: "true"` to the resource.

In the Kyverno policy a new `context` entry `resourceCache` will be added. Here is an example:

```yaml
context:
  - name: ingresses
    resourceCache:
      group: "apis/networking.k8s.io"
      version: "v1"
      kind: "ingresses"
      namespace: apps
      jmesPath: "ingresses | items[].spec.rules[].host"
```

This allows policy authors to declare what resource should be cached. 

The `group` and `version` are optional. If not specified, the preferred versions should be used.

An optional `namespace` can be used to only cache resources in the namespace, rather than across all namespaces which is the behavior is a namespace is not specified.

The JMESPath is also optional and is applied to add a resulting subset of the resource data to the rule context.

Note that Kyverno will only cache matching resources that have the label: `cache.kyverno.io/enabled: "true"`.

### Implementation

There are multiple ways to implement this feature:

#### Limited to known types

With this option, Kyverno will be able to cache types defined in the Kubernetes client set but will not be able to cache other custom resources.

As policies are created or modified, Kyverno will attempt to initialize informers for any `resourceCache` declaration. In case an informer cannot be initialized, an error will be returned.

During rule execution, Kyverno will add the resource data to the rule context.

#### Support for any resource type

With this option, Kyverno will not be able to use informers but instead use dynamic watches and maintain its cache.

This will be more involved but will allow caching of any custom resource. This approach can also allow finer-grained filters, in addition to labels, for what should be cached.

As with the informers-based implementation, during rule execution, Kyverno will add the resource data to the rule context.

### Drawbacks

1. API calls do not leverage caching by default. If needed, we can add a separate caching mechanism for API calls in the future.
2. Using `cache.kyverno.io/enabled: "true"` can cause issues when users forget to add them to the resource. 
   1. It is easy to miss a resource when adding the label. `apiCall` will return all the resources of the given type while `resourceCache` will return only those resources that have the label. In the case of 1-10k resources. 
   2. Users might not want to have a Kyverno-specific label in all their resources across all namespaces.
3. For the use case mentioned in [motivation](#motivation), we need to add a resource to the cache when the policies is applied, otherwise, when the resource is applied for the first time, it will fail because of the timeout like it currently does. This will take away the ability to have substitutions in `resourceCache` (e.g. `namespace: "{{request.namespace}}"`) and the `resourceCache` field will have to be static.

## Add caching to API calls 

To cache resources, the `apiCall` context entry can be extended to add a `cache` field:

```yaml
      context:
        - name: hosts
          apiCall:
            urlPath: "/apis/networking.k8s.io/v1/ingresses"
            jmesPath: "items[].spec.rules[].host"
            cache: true
```

However, coupling resource caching to API calls could be confusing for users.

# Prior Art

N/A

# Limitations

# Open Items

1. We may not be able to use a static Kubernetes client for all types, as the client set can include custom types, and dynamic clients may be resource-intensive. More research is needed to determine the best way to manage informers. The alternative is to use watches.
2. Typically informers are initialized on startup. This feature may require adding/deleting informers after startup.
3. All admission controller replicas will need to cache data. For the background controller and reports controller, the leader will need to cache data.


# CRD Changes (OPTIONAL)

Yes. A new context entry `resourceCache` will be added. The CRD will be defined in the implementation stage.