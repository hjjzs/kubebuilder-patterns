# Resource Deletion 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 安全删除、防止竞态条件、UID 检查

## 概述

Resource Deletion 模式通过 UID 检查和状态保存，确保在删除资源时不会误操作重建的资源，并在删除前保存重要状态信息。

## 快速开始

```go
// 安全删除 Pod（带 UID 检查）
func (r *DevboxReconciler) deletePod(ctx context.Context, devbox *Devbox, pod *corev1.Pod) error {
    logger := log.FromContext(ctx)
    originalPodUID := pod.UID  // 保存原始 UID
    
    // 1. 保存容器状态
    if len(pod.Status.ContainerStatuses) > 0 {
        err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
            latest := &Devbox{}
            if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
                return err
            }
            latest.Status.LastContainerStatus = pod.Status.ContainerStatuses[0]
            return r.Status().Update(ctx, latest)
        })
        if err != nil {
            return err
        }
    }
    
    // 2. 移除 Finalizer（带 UID 检查）
    err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latestPod := &corev1.Pod{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(pod), latestPod); err != nil {
            if apierrors.IsNotFound(err) {
                return nil  // 已删除
            }
            return err
        }
        
        // UID 检查：防止操作重建的 Pod
        if latestPod.UID != originalPodUID {
            logger.Info("pod UID changed, skip finalizer removal",
                "originalUID", originalPodUID,
                "currentUID", latestPod.UID)
            return nil
        }
        
        controllerutil.RemoveFinalizer(latestPod, FinalizerName)
        return r.Update(ctx, latestPod)
    })
    if err != nil {
        return err
    }
    
    // 3. 删除 Pod
    return r.Delete(ctx, pod,
        client.GracePeriodSeconds(0),
        client.PropagationPolicy(metav1.DeletePropagationBackground),
    )
}
```

---

## 1. UID 检查模式

### 1.1 为什么需要 UID 检查

**场景**：删除 Pod 的竞态条件

```
时间线:
T1: Controller 读取 Pod-A (UID: 111)
T2: 用户删除 Pod-A
T3: Kubelet 重建 Pod-A (UID: 222)
T4: Controller 尝试操作 Pod-A
    - 没有 UID 检查：操作新的 Pod-A (UID: 222) ❌
    - 有 UID 检查：跳过操作 ✅
```

### 1.2 UID 检查实现

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) deletePod(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    pod *corev1.Pod,
) error {
    logger := log.FromContext(ctx)
    originalPodUID := pod.UID  // 保存原始 UID
    
    // 移除 Finalizer
    err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latestPod := &corev1.Pod{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(pod), latestPod); err != nil {
            if apierrors.IsNotFound(err) {
                return nil  // Pod 已删除，成功
            }
            return err
        }
        
        // UID 检查
        if latestPod.UID != originalPodUID {
            logger.Info("pod UID changed, skip finalizer removal",
                "originalUID", originalPodUID,
                "currentUID", latestPod.UID,
                "pod", pod.Name)
            return nil  // 跳过操作
        }
        
        controllerutil.RemoveFinalizer(latestPod, FinalizerName)
        return r.Update(ctx, latestPod)
    })
    
    if err != nil {
        logger.Error(err, "failed to remove pod finalizer")
        return err
    }
    
    // 删除 Pod
    return r.Delete(ctx, pod,
        client.GracePeriodSeconds(0),
        client.PropagationPolicy(metav1.DeletePropagationBackground),
    )
}
```

---

## 2. 状态保存

### 2.1 保存容器状态

```go
// 删除前保存容器状态，用于调试和恢复
if len(pod.Status.ContainerStatuses) > 0 {
    err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
            return err
        }
        
        // 保存最后的容器状态
        latest.Status.LastContainerStatus = pod.Status.ContainerStatuses[0]
        
        return r.Status().Update(ctx, latest)
    })
    
    if err != nil {
        logger.Error(err, "failed to update devbox status")
        return err
    }
}
```

### 2.2 保存退出信息

```go
type DevboxStatus struct {
    // LastContainerStatus is the last observed container status
    LastContainerStatus corev1.ContainerStatus `json:"lastContainerStatus,omitempty"`
}

// 用户可以查看容器的退出原因
// kubectl get devbox my-devbox -o jsonpath='{.status.lastContainerStatus.state.terminated.reason}'
// 输出: OOMKilled, Error, Completed 等
```

---

## 3. 删除选项

### 3.1 GracePeriodSeconds

```go
// 立即删除（0 秒优雅期）
r.Delete(ctx, pod, client.GracePeriodSeconds(0))

// 默认优雅期（30 秒）
r.Delete(ctx, pod)

// 自定义优雅期
r.Delete(ctx, pod, client.GracePeriodSeconds(60))
```

### 3.2 PropagationPolicy

```go
// Background: 立即删除，子资源后台删除
r.Delete(ctx, pod, 
    client.PropagationPolicy(metav1.DeletePropagationBackground))

// Foreground: 等待所有子资源删除后才删除
r.Delete(ctx, pod,
    client.PropagationPolicy(metav1.DeletePropagationForeground))

// Orphan: 删除但不删除子资源
r.Delete(ctx, pod,
    client.PropagationPolicy(metav1.DeletePropagationOrphan))
```

---

## 4. 批量删除

### 4.1 按 Label 批量删除

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) deleteResourcesByLabels(
    ctx context.Context,
    obj client.Object,
    namespace string,
    labels map[string]string,
) error {
    return client.IgnoreNotFound(r.DeleteAllOf(ctx, obj,
        client.InNamespace(namespace),
        client.MatchingLabels(labels),
    ))
}

// 使用
recLabels := map[string]string{
    "app.kubernetes.io/name": devbox.Name,
}

// 删除所有匹配的 Pod
if err := r.deleteResourcesByLabels(ctx, &corev1.Pod{}, devbox.Namespace, recLabels); err != nil {
    return err
}

// 删除所有匹配的 Service
if err := r.deleteResourcesByLabels(ctx, &corev1.Service{}, devbox.Namespace, recLabels); err != nil {
    return err
}
```

### 4.2 并发删除

```go
func (r *DevboxReconciler) cleanupResources(ctx context.Context, devbox *Devbox, recLabels map[string]string) error {
    var wg sync.WaitGroup
    errCh := make(chan error, 3)
    
    // 并发删除多种资源
    resources := []client.Object{
        &corev1.Pod{},
        &corev1.Service{},
        &corev1.Secret{},
    }
    
    for _, obj := range resources {
        wg.Add(1)
        go func(resource client.Object) {
            defer wg.Done()
            if err := r.deleteResourcesByLabels(ctx, resource, devbox.Namespace, recLabels); err != nil {
                errCh <- err
            }
        }(obj)
    }
    
    wg.Wait()
    close(errCh)
    
    // 收集错误
    for err := range errCh {
        return err
    }
    
    return nil
}
```

---

## 5. 最佳实践

### 5.1 ✅ 始终使用 UID 检查

```go
// ✅ 正确：检查 UID
originalUID := pod.UID
// ... 执行操作 ...
if latestPod.UID != originalUID {
    return nil  // 跳过
}

// ❌ 错误：不检查 UID
// 可能操作到重建的资源
```

### 5.2 ✅ 删除前保存状态

```go
// ✅ 正确：保存重要信息
if len(pod.Status.ContainerStatuses) > 0 {
    devbox.Status.LastContainerStatus = pod.Status.ContainerStatuses[0]
    r.Status().Update(ctx, devbox)
}

// 然后删除
r.Delete(ctx, pod)
```

### 5.3 ✅ 使用 IgnoreNotFound

```go
// ✅ 正确：忽略 NotFound 错误
if err := r.Delete(ctx, pod); err != nil {
    return client.IgnoreNotFound(err)
}

// ❌ 错误：NotFound 也返回错误
if err := r.Delete(ctx, pod); err != nil {
    return err
}
```

### 5.4 ✅ 先移除 Finalizer，再删除

```go
// ✅ 正确：先移除 Finalizer
controllerutil.RemoveFinalizer(pod, FinalizerName)
r.Update(ctx, pod)

// 然后删除
r.Delete(ctx, pod)

// ❌ 错误：先删除，Finalizer 会阻止删除
r.Delete(ctx, pod)  // 卡住，因为有 Finalizer
```

---

## 6. 总结

### 核心要点

1. ✅ **UID 检查** - 防止操作重建的资源
2. ✅ **保存状态** - 删除前保存重要信息
3. ✅ **IgnoreNotFound** - 资源已删除不算错误
4. ✅ **先移除 Finalizer** - 再删除资源
5. ✅ **使用删除选项** - GracePeriodSeconds, PropagationPolicy

### 适用场景

| 场景 | 是否使用 UID 检查 |
|------|----------------|
| 删除可能重建的资源 | ✅ 是 |
| 删除一次性资源 | ❌ 否 |
| 批量删除 | ⚠️ 视情况 |

### 与其他模式的配合

- **Finalizer**: 在 Finalizer 清理逻辑中使用
- **RetryOnConflict**: 移除 Finalizer 时使用
- **Event Recording**: 记录删除操作

### 参考资源

- [Sealos Devbox Controller](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
- [Kubernetes Delete Options](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/delete-options/)
