---
name: kubebuilder-patterns
description: Kubebuilder/controller-runtime design patterns guide for building production-ready Kubernetes controllers. Use when developing, reviewing, or debugging Kubernetes operators and controllers, implementing reconciliation logic, managing resource lifecycle, handling concurrent updates, integrating external services, or following Kubernetes controller best practices.
---

# Kubebuilder Design Patterns

Production-validated patterns from Sealos controllers (Account, Devbox, User, Terminal, ObjectStorage, License).

## Pattern Selection Guide

| Scenario | Pattern |
|----------|---------|
| Update resource status | [RetryOnConflict](references/01-retry-on-conflict.md) |
| Create child resources | [CreateOrUpdate](references/02-create-or-update.md) + [Owner Reference](references/13-owner-reference.md) |
| Cleanup before deletion | [Finalizer](references/03-finalizer.md) |
| Reduce unnecessary Reconciles | [Custom Predicate](references/04-custom-predicate.md) |
| Optimize List queries | [Field Indexer](references/05-field-indexer.md) |
| Handle large resource counts | [Concurrent Reconcile](references/06-concurrent-reconcile.md) |
| Multi-step operations | [Pipeline](references/07-pipeline.md) |
| One-time async operations | [Request-Driven](references/08-request-driven.md) |
| Aggregate status from multiple sources | [Phase Derivation](references/09-phase-derivation.md) |
| External system integration | [External Service](references/10-external-service.md) |
| Periodic background tasks | [Scheduled Task](references/11-scheduled-task.md) |
| Audit and diagnostics | [Event Recording](references/12-event-recording.md) |
| Auto-expire resources | [TTL/Expiration](references/14-ttl-expiration.md) |
| Safe resource deletion | [Resource Deletion](references/15-resource-deletion.md) |
| Environment configuration | [Configuration Loading](references/16-configuration.md) |
| Resource identification | [Recommended Labels](references/17-labels.md) |
| Error handling | [Error Handling](references/18-error-handling.md) |

## Pattern Categories

### Core Patterns (Essential)
1. **RetryOnConflict** - Handle optimistic locking conflicts with automatic retry
2. **CreateOrUpdate** - Declarative resource management for child resources
3. **Finalizer** - Pre-deletion hooks for cleanup and cascade delete
4. **Custom Predicate** - Event filtering to reduce unnecessary reconciliations

### Performance Patterns
5. **Field Indexer** - Cache indexing for O(1) List queries
6. **Concurrent Reconcile** - Parallel processing with MaxConcurrentReconciles

### Architecture Patterns
7. **Pipeline** - Multi-step orchestration with Context-based state passing
8. **Request-Driven** - Command-style operations via Request CRDs
9. **Phase Derivation** - Status aggregation from multiple signal sources

### Integration Patterns
10. **External Service** - MinIO, database, message queue integration
11. **Scheduled Task** - Background periodic tasks via manager.Runnable
12. **Event Recording** - Kubernetes Events for audit and observability

### Resource Management Patterns
13. **Owner Reference** - Automatic cascade deletion
14. **TTL/Expiration** - Auto-cleanup for temporary resources
15. **Resource Deletion** - Safe deletion with UID checks

### Engineering Practices
16. **Configuration Loading** - Environment-based configuration
17. **Recommended Labels** - Kubernetes standard labels
18. **Error Handling** - Error classification, retry strategies, recovery

## Quick Reference

### Reconcile Structure Template

```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    // 1. Fetch resource
    obj := &MyResource{}
    if err := r.Get(ctx, req.NamespacedName, obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. Handle deletion (Finalizer pattern)
    if !obj.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, obj)
    }
    
    // 3. Ensure Finalizer
    if err := r.ensureFinalizer(ctx, obj); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. Execute Pipeline
    for _, fn := range r.reconcilePipeline {
        if err := fn(ctx, obj); err != nil {
            logger.Error(err, "pipeline step failed")
            r.Recorder.Eventf(obj, corev1.EventTypeWarning, "SyncFailed", "%v", err)
            return ctrl.Result{}, err
        }
    }
    
    // 5. Update Phase (Phase Derivation pattern)
    return r.syncPhase(ctx, obj)
}
```

### Common Pattern Combinations

| Use Case | Patterns |
|----------|----------|
| Standard controller | Finalizer + CreateOrUpdate + Owner Reference + Event Recording |
| High-volume controller | Custom Predicate + Field Indexer + Concurrent Reconcile |
| Complex workflows | Pipeline + Phase Derivation + Error Handling |
| External integrations | External Service + Scheduled Task + Configuration Loading |

## Usage

Read the relevant pattern reference file based on your implementation needs. Each pattern file contains:
- Problem background and motivation
- Implementation with code examples
- Best practices and common errors
- Real-world cases from Sealos controllers
