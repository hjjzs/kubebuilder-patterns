# Finalizer 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 资源清理、级联删除、外部资源管理

## 概述

Finalizer 是 Kubernetes 提供的资源删除前置钩子机制。它确保在资源被真正删除前，控制器有机会执行清理逻辑（如删除外部资源、清理子资源、释放配额等）。

## 目录

1. [工作原理](#1-工作原理)
2. [两种实现方式](#2-两种实现方式)
3. [使用方法](#3-使用方法)
4. [最佳实践](#4-最佳实践)
5. [常见错误](#5-常见错误)
6. [实际案例](#6-实际案例)

---

## 1. 工作原理

### 1.1 删除流程

```
用户执行 kubectl delete devbox my-devbox
         │
         ▼
┌────────────────────────────────────────────────┐
│ API Server 检查 Finalizers                     │
│ metadata.finalizers: ["devbox.sealos.io/finalizer"] │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 设置 metadata.deletionTimestamp = 现在时间     │
│ 资源进入 "删除中" 状态，但不会真正删除         │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 触发 Controller Reconcile                      │
│ 检测到 deletionTimestamp != nil                │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 执行清理逻辑                                    │
│ - 删除 Pod、Service 等子资源                   │
│ - 清理外部资源（数据库记录、对象存储等）        │
│ - 释放配额                                      │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 移除 Finalizer                                  │
│ metadata.finalizers: []                         │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ API Server 真正删除资源                         │
└────────────────────────────────────────────────┘
```

### 1.2 关键字段

```yaml
apiVersion: devbox.sealos.io/v1alpha2
kind: Devbox
metadata:
  name: my-devbox
  namespace: user-ns
  finalizers:
  - devbox.sealos.io/finalizer  # Finalizer 列表
  deletionTimestamp: "2026-01-23T10:00:00Z"  # 删除时间戳（设置后表示正在删除）
  deletionGracePeriodSeconds: 30  # 优雅删除时间
spec:
  # ...
```

---

## 2. 两种实现方式

### 2.1 方式一：封装的 Finalizer Helper

Sealos User 控制器使用封装的 Helper：

```go
// controllers/user/controllers/helper/finalizer/finalizer.go
type Finalizer struct {
    client        client.Client
    finalizerName string
}

func NewFinalizer(client client.Client, name string) *Finalizer {
    return &Finalizer{
        client:        client,
        finalizerName: name,
    }
}

func (f *Finalizer) AddFinalizer(ctx context.Context, obj client.Object) (bool, error) {
    if obj.GetDeletionTimestamp() == nil || obj.GetDeletionTimestamp().IsZero() {
        controllerutil.AddFinalizer(obj, f.finalizerName)
        return true, f.updateFinalizers(ctx, obj)
    }
    return false, nil
}

func (f *Finalizer) RemoveFinalizer(
    ctx context.Context,
    obj client.Object,
    cleanup func(context.Context, client.Object) error,
) (bool, error) {
    if !obj.GetDeletionTimestamp().IsZero() && 
       controllerutil.ContainsFinalizer(obj, f.finalizerName) {
        // 执行清理逻辑
        if err := cleanup(ctx, obj); err != nil {
            return true, err
        }
        
        // 移除 Finalizer
        controllerutil.RemoveFinalizer(obj, f.finalizerName)
        return true, f.updateFinalizers(ctx, obj)
    }
    return false, nil
}

func (f *Finalizer) updateFinalizers(ctx context.Context, obj client.Object) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &unstructured.Unstructured{}
        latest.SetGroupVersionKind(obj.GetObjectKind().GroupVersionKind())
        
        if err := f.client.Get(ctx, client.ObjectKeyFromObject(obj), latest); err != nil {
            return err
        }
        
        latest.SetFinalizers(obj.GetFinalizers())
        return f.client.Update(ctx, latest)
    })
}
```

**使用示例**：

```go
type UserReconciler struct {
    client.Client
    finalizer *finalizer.Finalizer
}

func (r *UserReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.finalizer = finalizer.NewFinalizer(r.Client, "sealos.io/user.finalizers")
    // ...
}

func (r *UserReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    user := &userv1.User{}
    if err := r.Get(ctx, req.NamespacedName, user); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 处理删除
    if ok, err := r.finalizer.RemoveFinalizer(ctx, user, func(ctx context.Context, obj client.Object) error {
        return r.cleanupUser(ctx, obj.(*userv1.User))
    }); ok {
        return ctrl.Result{}, err
    }
    
    // 添加 Finalizer
    if _, err := r.finalizer.AddFinalizer(ctx, user); err != nil {
        return ctrl.Result{}, err
    }
    
    // 正常 Reconcile 逻辑
    return r.reconcile(ctx, user)
}
```

### 2.2 方式二：内联 Finalizer 处理

Devbox 控制器使用内联方式：

```go
const FinalizerName = "devbox.sealos.io/finalizer"

func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 处理删除
    if !devbox.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(devbox, FinalizerName) {
            // 执行清理逻辑
            if err := r.cleanupDevbox(ctx, devbox); err != nil {
                return ctrl.Result{}, err
            }
            
            // 移除 Finalizer
            if controllerutil.RemoveFinalizer(devbox, FinalizerName) {
                if err := r.Update(ctx, devbox); err != nil {
                    return ctrl.Result{}, err
                }
            }
        }
        return ctrl.Result{}, nil
    }
    
    // 添加 Finalizer
    if !controllerutil.ContainsFinalizer(devbox, FinalizerName) {
        if controllerutil.AddFinalizer(devbox, FinalizerName) {
            if err := r.Update(ctx, devbox); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{Requeue: true}, nil  // 重新入队，确保 Finalizer 生效
    }
    
    // 正常 Reconcile 逻辑
    return r.reconcile(ctx, devbox)
}
```

---

## 3. 使用方法

### 3.1 添加 Finalizer（使用 RetryOnConflict）

```go
// controllers/license/internal/controller/license_controller.go
func (r *LicenseReconciler) ensureFinalizer(ctx context.Context, req ctrl.Request) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &licensev1.License{}
        if err := r.Get(ctx, req.NamespacedName, latest); err != nil {
            return err
        }
        
        if controllerutil.AddFinalizer(latest, licenseFinalizer) {
            return r.Update(ctx, latest)
        }
        return nil  // 已存在，无需更新
    })
}
```

### 3.2 移除 Finalizer（使用 RetryOnConflict）

```go
func (r *LicenseReconciler) removeFinalizer(ctx context.Context, req ctrl.Request) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &licensev1.License{}
        if err := r.Get(ctx, req.NamespacedName, latest); err != nil {
            return client.IgnoreNotFound(err)
        }
        
        if controllerutil.RemoveFinalizer(latest, licenseFinalizer) {
            return r.Update(ctx, latest)
        }
        return nil
    })
}
```

### 3.3 清理子资源

```go
func (r *DevboxReconciler) cleanupDevbox(ctx context.Context, devbox *devboxv1alpha2.Devbox) error {
    logger := log.FromContext(ctx)
    
    recLabels := map[string]string{
        "app.kubernetes.io/name": devbox.Name,
    }
    
    // 删除 Pod
    if err := r.deleteResourcesByLabels(ctx, &corev1.Pod{}, devbox.Namespace, recLabels); err != nil {
        logger.Error(err, "failed to delete pods")
        return err
    }
    
    // 删除 Service
    if err := r.deleteResourcesByLabels(ctx, &corev1.Service{}, devbox.Namespace, recLabels); err != nil {
        logger.Error(err, "failed to delete services")
        return err
    }
    
    // 删除 Secret
    if err := r.deleteResourcesByLabels(ctx, &corev1.Secret{}, devbox.Namespace, recLabels); err != nil {
        logger.Error(err, "failed to delete secrets")
        return err
    }
    
    logger.Info("devbox cleanup completed")
    return nil
}

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
```

### 3.4 清理外部资源

```go
// controllers/objectstorage/controllers/objectstoragebucket_controller.go
func (r *ObjectStorageBucketReconciler) cleanupBucket(
    ctx context.Context,
    bucket *objectstoragev1.ObjectStorageBucket,
) error {
    bucketName := buildBucketName(bucket.Name, bucket.Namespace)
    
    // 1. 清空 Bucket 中的所有对象
    objects := r.OSClient.ListObjects(ctx, bucketName, minio.ListObjectsOptions{
        Recursive: true,
    })
    for object := range objects {
        if err := r.OSClient.RemoveObject(ctx, bucketName, object.Key, minio.RemoveObjectOptions{}); err != nil {
            return fmt.Errorf("failed to remove object %s: %w", object.Key, err)
        }
    }
    
    // 2. 删除 Bucket
    if err := r.OSClient.RemoveBucket(ctx, bucketName); err != nil {
        return fmt.Errorf("failed to remove bucket: %w", err)
    }
    
    // 3. 删除 Service Account
    serviceAccountName := buildSAName(bucketName)
    if err := r.OSAdminClient.DeleteServiceAccount(ctx, serviceAccountName); err != nil {
        return fmt.Errorf("failed to delete service account: %w", err)
    }
    
    return nil
}
```

---

## 4. 最佳实践

### 4.1 ✅ 使用唯一的 Finalizer 名称

```go
// ✅ 正确：使用域名格式
const FinalizerName = "devbox.sealos.io/finalizer"
const LicenseFinalizer = "license.sealos.io/finalizer"

// ❌ 错误：通用名称可能冲突
const FinalizerName = "finalizer"
```

### 4.2 ✅ 清理逻辑必须幂等

```go
// ✅ 正确：幂等的清理逻辑
func (r *DevboxReconciler) cleanupDevbox(ctx context.Context, devbox *devboxv1alpha2.Devbox) error {
    // 使用 IgnoreNotFound 处理资源已删除的情况
    if err := r.deleteResourcesByLabels(ctx, &corev1.Pod{}, devbox.Namespace, labels); err != nil {
        return client.IgnoreNotFound(err)
    }
    return nil
}

// ❌ 错误：非幂等的清理逻辑
func (r *DevboxReconciler) cleanupDevbox(ctx context.Context, devbox *devboxv1alpha2.Devbox) error {
    // 如果资源已删除，返回错误会导致 Finalizer 无法移除
    return r.deleteResourcesByLabels(ctx, &corev1.Pod{}, devbox.Namespace, labels)
}
```

### 4.3 ✅ 添加 Finalizer 后立即 Requeue

```go
// ✅ 正确：添加 Finalizer 后 Requeue
if !controllerutil.ContainsFinalizer(devbox, FinalizerName) {
    if controllerutil.AddFinalizer(devbox, FinalizerName) {
        if err := r.Update(ctx, devbox); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{Requeue: true}, nil  // 重新入队
}

// 继续正常逻辑
return r.reconcile(ctx, devbox)
```

### 4.4 ✅ 使用 RetryOnConflict 处理并发

```go
// ✅ 正确：使用 RetryOnConflict
err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    if err := r.Get(ctx, key, latest); err != nil {
        return client.IgnoreNotFound(err)
    }
    
    if controllerutil.RemoveFinalizer(latest, FinalizerName) {
        return r.Update(ctx, latest)
    }
    return nil
})
```

### 4.5 ✅ 记录清理日志

```go
func (r *DevboxReconciler) cleanupDevbox(ctx context.Context, devbox *devboxv1alpha2.Devbox) error {
    logger := log.FromContext(ctx)
    logger.Info("starting devbox cleanup", "devbox", devbox.Name)
    
    if err := r.deletePods(ctx, devbox); err != nil {
        logger.Error(err, "failed to delete pods")
        return err
    }
    logger.Info("pods deleted")
    
    if err := r.deleteServices(ctx, devbox); err != nil {
        logger.Error(err, "failed to delete services")
        return err
    }
    logger.Info("services deleted")
    
    logger.Info("devbox cleanup completed")
    return nil
}
```

---

## 5. 常见错误

### 5.1 ❌ 清理逻辑失败导致 Finalizer 无法移除

```go
// ❌ 错误：清理失败会导致资源永远无法删除
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if !devbox.DeletionTimestamp.IsZero() {
        // 如果清理失败，Finalizer 无法移除，资源永远卡在删除中
        if err := r.cleanupDevbox(ctx, devbox); err != nil {
            return ctrl.Result{}, err  // ❌ 返回错误
        }
        
        controllerutil.RemoveFinalizer(devbox, FinalizerName)
        return ctrl.Result{}, r.Update(ctx, devbox)
    }
    
    return r.reconcile(ctx, devbox)
}

// ✅ 正确：添加重试和超时机制
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if !devbox.DeletionTimestamp.IsZero() {
        // 尝试清理
        if err := r.cleanupDevbox(ctx, devbox); err != nil {
            logger.Error(err, "cleanup failed, will retry")
            
            // 检查是否超时（超过 5 分钟强制移除 Finalizer）
            if time.Since(devbox.DeletionTimestamp.Time) > 5*time.Minute {
                logger.Warn("cleanup timeout, forcing finalizer removal")
            } else {
                return ctrl.Result{RequeueAfter: 30 * time.Second}, nil  // 重试
            }
        }
        
        controllerutil.RemoveFinalizer(devbox, FinalizerName)
        return ctrl.Result{}, r.Update(ctx, devbox)
    }
    
    return r.reconcile(ctx, devbox)
}
```

### 5.2 ❌ 忘记检查 DeletionTimestamp

```go
// ❌ 错误：没有检查 DeletionTimestamp，正常逻辑会继续执行
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // ❌ 没有处理删除逻辑
    
    // 正常逻辑继续执行，可能创建新资源
    return r.reconcile(ctx, devbox)
}

// ✅ 正确：优先处理删除
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // ✅ 优先处理删除
    if !devbox.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, devbox)
    }
    
    return r.reconcile(ctx, devbox)
}
```

### 5.3 ❌ 多个 Finalizer 的顺序问题

```go
// ❌ 错误：多个 Finalizer 可能导致死锁
metadata:
  finalizers:
  - controller-a/finalizer  # Controller A 等待 Controller B 清理
  - controller-b/finalizer  # Controller B 等待 Controller A 清理

// ✅ 正确：确保 Finalizer 独立，不相互依赖
// Controller A 只清理自己管理的资源
// Controller B 只清理自己管理的资源
```

---

## 6. 实际案例

### 6.1 案例 1：License 控制器的保护 Finalizer

License 控制器使用两个 Finalizer：
- `license.sealos.io/finalizer`：正常清理
- `license.sealos.io/protection`：防止删除默认 License

```go
// controllers/license/internal/controller/license_controller.go
const (
    licenseFinalizer           = "license.sealos.io/finalizer"
    licenseProtectionFinalizer = "license.sealos.io/protection"
)

func (r *LicenseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    license := &licensev1.License{}
    if err := r.Get(ctx, req.NamespacedName, license); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if license.DeletionTimestamp.IsZero() {
        // 添加 Finalizer
        if err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
            latest := &licensev1.License{}
            if err := r.Get(ctx, req.NamespacedName, latest); err != nil {
                return err
            }
            changed := controllerutil.AddFinalizer(latest, licenseFinalizer)
            if !changed {
                return nil
            }
            return r.Update(ctx, latest)
        }); err != nil {
            return ctrl.Result{}, err
        }
    } else {
        // 处理删除
        
        // 检查保护 Finalizer
        if controllerutil.ContainsFinalizer(license, licenseProtectionFinalizer) && isDefaultLicense(license) {
            r.Logger.Info("deletion blocked by protection finalizer", "license", req.NamespacedName)
            return ctrl.Result{}, nil  // 阻止删除
        }
        
        // 移除普通 Finalizer
        if err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
            latest := &licensev1.License{}
            if err := r.Get(ctx, req.NamespacedName, latest); err != nil {
                return client.IgnoreNotFound(err)
            }
            if controllerutil.RemoveFinalizer(latest, licenseFinalizer) {
                return r.Update(ctx, latest)
            }
            return nil
        }); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return ctrl.Result{}, nil
}

func isDefaultLicense(license *licensev1.License) bool {
    return license.Name == "default-license"
}
```

### 6.2 案例 2：Devbox 控制器的 Pod 清理

Devbox 需要在删除前清理 Pod，并更新状态：

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) deletePod(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    pod *corev1.Pod,
) error {
    logger := log.FromContext(ctx)
    originalPodUID := pod.UID
    
    // 1. 更新 Devbox 状态（保存容器状态）
    if len(pod.Status.ContainerStatuses) > 0 {
        err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
            latest := &devboxv1alpha2.Devbox{}
            if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
                return err
            }
            latest.Status.LastContainerStatus = pod.Status.ContainerStatuses[0]
            return r.Status().Update(ctx, latest)
        })
        if err != nil {
            logger.Error(err, "failed to update devbox status")
            return err
        }
    }
    
    // 2. 移除 Pod 的 Finalizer（带 UID 检查）
    err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latestPod := &corev1.Pod{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(pod), latestPod); err != nil {
            if apierrors.IsNotFound(err) {
                return nil  // Pod 已删除
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

## 7. 总结

### 核心要点

1. ✅ **清理逻辑必须幂等** - 多次执行结果相同
2. ✅ **使用 RetryOnConflict** - 处理并发冲突
3. ✅ **添加 Finalizer 后 Requeue** - 确保生效
4. ✅ **处理清理超时** - 避免资源永久卡住
5. ✅ **记录详细日志** - 便于排查问题

### 适用场景

| 场景 | 是否使用 Finalizer |
|------|------------------|
| 清理子资源 | ✅ 是（如果没有 Owner Reference） |
| 清理外部资源 | ✅ 是 |
| 释放配额 | ✅ 是 |
| 删除数据库记录 | ✅ 是 |
| 通知外部系统 | ✅ 是 |
| 有 Owner Reference 的子资源 | ❌ 否（自动级联删除） |

### 与其他模式的配合

- **Owner Reference 模式**：有 Owner Reference 的资源会自动删除，无需 Finalizer
- **RetryOnConflict 模式**：添加/移除 Finalizer 时必须使用
- **External Service Integration 模式**：清理外部资源时使用 Finalizer

### 参考资源

- [Kubernetes Finalizers 文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/)
- [Using Finalizers](https://book.kubebuilder.io/reference/using-finalizers.html)
- [Sealos Devbox Controller 实现](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
