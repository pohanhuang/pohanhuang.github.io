---
title: "Debugging Controller OOM When Cache Grows Unbounded"
type: docs
---

## The Problem

The [Kubewarden sbomscanner](https://github.com/kubewarden/sbomscanner) controller was **OOMKilled** in production. Memory usage grew unbounded as the cluster scaled.

[GitHub Issue #438](https://github.com/kubewarden/sbomscanner/issues/438) tracked how user observed that the `sbombastic-controller` pod was consuming significantly more memory than other pods in the cluster, eventually hitting resource limits and being terminated by Kubernetes.

## Root Cause: Unbounded Cache Growth
Controller-runtime caches **every watched resource in memory**:

1. **Resources with reconcilers** → full cache
2. **Resources with indexers** → full cache (often overlooked)
3. **Large fields you don't need** → still cached

Example: VulnerabilityReport reconciler needs to query Images by reference, so it creates an indexer:
```go
mgr.GetFieldIndexer().IndexField(ctx, &VulnerabilityReport{},
    ".spec.image", indexFunc)
```

## The Solution: Field Stripping

The fix, implemented in [PR #444](https://github.com/kubewarden/sbomscanner/pull/444), reduces memory usage by **stripping unnecessary fields** from cached resources before they enter the cache.

The solution removes three types of fields:

1. **Managed Fields**: `metadata.managedFields` - Kubernetes tracks field managers here, but controllers typically don't need this for reconciliation
2. **Data Fields from Image resources**: Which used for indexing to help search the VulnerabilityReport resources
3. **Data Fields from VulnerabilityReport resources**: Similar to Image resources, these can contain large payloads

### Why This Works

1. **Managed fields accumulate**: Every update adds entries to `managedFields`. For frequently updated resources, this can grow to hundreds of KB per resource
2. **Data fields are large**: Image manifests and vulnerability reports can contain MBs of data
3. **Controllers don't need everything**: Reconciliation logic typically only needs spec fields and basic metadata

## Key Takeaways

1. **Controller caches can grow unbounded**: Without limits, watching many resources or large resources can cause OOM
2. **Indexers cache too**: Creating an index on a field caches the entire resource
3. **Strip early**: Transform before cache, not during reconcile

## References

- [Kubewarden sbomscanner Issue #438](https://github.com/kubewarden/sbomscanner/issues/438) - Original OOM issue
- [Kubewarden sbomscanner PR #444](https://github.com/kubewarden/sbomscanner/pull/444) - Memory optimization fix
- [Controller-Runtime Cache Documentation](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache)
- [Kubernetes Managed Fields](https://kubernetes.io/docs/reference/using-api/server-side-apply/#field-management)
