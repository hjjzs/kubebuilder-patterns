# Error Handling 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 错误处理、重试策略、故障恢复

## 概述

Error Handling 模式提供了 Kubernetes 控制器中错误处理的最佳实践，包括错误分类、重试策略、日志记录和用户反馈。

## 快速开始

```go
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        // 1. 忽略 NotFound 错误
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. 执行操作
    if err := r.syncPod(ctx, devbox); err != nil {
        // 3. 记录错误日志
        logger.Error(err, "failed to sync pod", "devbox", devbox.Name)
        
        // 4. 记录 Event
        r.Recorder.Eventf(devbox, corev1.EventTypeWarning, 
            "SyncFailed", "Failed to sync pod: %v", err)
        
        // 5. 更新 Status
        devbox.Status.Phase = devboxv1alpha2.DevboxPhaseError
        if updateErr := r.Status().Update(ctx, devbox); updateErr != nil {
            logger.Error(updateErr, "failed to update status")
        }
        
        // 6. 返回错误（触发重试）
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

---

## 1. 错误分类

### 1.1 可忽略的错误

```go
// NotFound 错误：资源不存在
if err := r.Get(ctx, key, obj); err != nil {
    return ctrl.Result{}, client.IgnoreNotFound(err)
}

// AlreadyExists 错误：资源已存在
if err := r.Create(ctx, pod); err != nil {
    if !apierrors.IsAlreadyExists(err) {
        return ctrl.Result{}, err
    }
    // 已存在，继续处理
}

// Conflict 错误：由 RetryOnConflict 处理
err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    // ...
})
// RetryOnConflict 会自动重试 Conflict 错误
```

### 1.2 临时错误 vs 永久错误

```go
func isTemporaryError(err error) bool {
    // 临时错误：可以重试
    return apierrors.IsServerTimeout(err) ||
           apierrors.IsTimeout(err) ||
           apierrors.IsTooManyRequests(err) ||
           apierrors.IsServiceUnavailable(err) ||
           apierrors.IsInternalError(err)
}

func isPermanentError(err error) bool {
    // 永久错误：重试无意义
    return apierrors.IsInvalid(err) ||
           apierrors.IsForbidden(err) ||
           apierrors.IsUnauthorized(err) ||
           apierrors.IsNotFound(err)
}

// 使用
if err := r.syncPod(ctx, devbox); err != nil {
    if isTemporaryError(err) {
        // 临时错误：延迟重试
        logger.Info("temporary error, will retry", "error", err)
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
    }
    
    if isPermanentError(err) {
        // 永久错误：不重试，记录错误
        logger.Error(err, "permanent error")
        r.Recorder.Event(devbox, corev1.EventTypeWarning, "PermanentError", err.Error())
        return ctrl.Result{}, nil  // 不返回错误，避免无限重试
    }
    
    // 其他错误：正常重试
    return ctrl.Result{}, err
}
```

---

## 2. 重试策略

### 2.1 立即重试

```go
// 返回错误，立即重新入队
return ctrl.Result{}, err
```

### 2.2 延迟重试

```go
// 延迟 30 秒后重试
return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
```

### 2.3 不重试

```go
// 永久错误，不重试
logger.Error(err, "permanent error")
return ctrl.Result{}, nil  // 不返回错误
```

### 2.4 指数退避

```go
// 使用 RateLimiter 实现指数退避
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        WithOptions(controller.Options{
            RateLimiter: workqueue.NewItemExponentialFailureRateLimiter(
                time.Second,      // BaseDelay: 1s
                1000*time.Second, // MaxDelay: 1000s
            ),
        }).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}

// 重试延迟：1s, 2s, 4s, 8s, 16s, ..., 1000s
```

---

## 3. 错误日志

### 3.1 结构化日志

```go
// ✅ 正确：结构化日志
logger.Error(err, "failed to sync pod",
    "devbox", devbox.Name,
    "namespace", devbox.Namespace,
    "phase", devbox.Status.Phase,
    "podName", podName)

// ❌ 错误：字符串拼接
logger.Error(err, fmt.Sprintf("failed to sync pod %s/%s", devbox.Namespace, devbox.Name))
```

### 3.2 日志级别

```go
// V(0): 重要错误（默认级别）
logger.Error(err, "failed to create pod")

// V(1): 详细信息
logger.V(1).Info("attempting to update status", "devbox", devbox.Name)

// V(2): 调试信息
logger.V(2).Info("pod status", "phase", pod.Status.Phase, "conditions", pod.Status.Conditions)
```

---

## 4. 用户反馈

### 4.1 更新 Status

```go
// ✅ 正确：在 Status 中反馈错误
if err := r.syncPod(ctx, devbox); err != nil {
    devbox.Status.Phase = devboxv1alpha2.DevboxPhaseError
    devbox.Status.Message = err.Error()
    
    if updateErr := r.Status().Update(ctx, devbox); updateErr != nil {
        logger.Error(updateErr, "failed to update status")
    }
    
    return ctrl.Result{}, err
}
```

### 4.2 使用 Condition

```go
type Condition struct {
    Type               ConditionType
    Status             corev1.ConditionStatus
    LastTransitionTime metav1.Time
    Reason             string
    Message            string
}

// 更新 Condition
func (r *UserReconciler) syncNamespace(ctx context.Context, user *User) context.Context {
    condition := &Condition{
        Type:   ConditionTypeNamespace,
        Status: corev1.ConditionTrue,
        Reason: "NamespaceReady",
    }
    
    if err := r.createNamespace(ctx, user); err != nil {
        condition.Status = corev1.ConditionFalse
        condition.Reason = "CreateFailed"
        condition.Message = err.Error()
    }
    
    user.Status.Conditions = updateCondition(user.Status.Conditions, condition)
    return ctx
}
```

### 4.3 记录 Event

```go
// ✅ 正确：记录详细的 Event
if err := r.syncPod(ctx, devbox); err != nil {
    r.Recorder.Eventf(devbox, corev1.EventTypeWarning,
        "SyncPodFailed",
        "Failed to sync pod: %v", err)
    return ctrl.Result{}, err
}

r.Recorder.Event(devbox, corev1.EventTypeNormal,
    "PodSynced", "Pod synced successfully")
```

---

## 5. 错误包装

### 5.1 添加上下文信息

```go
// ✅ 正确：包装错误，添加上下文
if err := r.syncPod(ctx, devbox); err != nil {
    return ctrl.Result{}, fmt.Errorf("failed to sync pod for devbox %s/%s: %w",
        devbox.Namespace, devbox.Name, err)
}

// ❌ 错误：直接返回原始错误
if err := r.syncPod(ctx, devbox); err != nil {
    return ctrl.Result{}, err  // 缺少上下文
}
```

### 5.2 错误链

```go
import "errors"

// 创建错误
var ErrPodNotReady = errors.New("pod is not ready")

// 包装错误
if !isPodReady(pod) {
    return fmt.Errorf("devbox %s: %w", devbox.Name, ErrPodNotReady)
}

// 判断错误类型
if errors.Is(err, ErrPodNotReady) {
    // 特殊处理
}
```

---

## 6. 实际案例

### 案例 1：Devbox 控制器的完整错误处理

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        // 忽略 NotFound
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 处理删除
    if !devbox.DeletionTimestamp.IsZero() {
        if err := r.handleDeletion(ctx, devbox); err != nil {
            logger.Error(err, "failed to handle deletion")
            r.Recorder.Eventf(devbox, corev1.EventTypeWarning,
                "DeletionFailed", "Failed to handle deletion: %v", err)
            return ctrl.Result{}, err
        }
        return ctrl.Result{}, nil
    }
    
    // 添加 Finalizer
    if err := r.ensureFinalizer(ctx, devbox); err != nil {
        logger.Error(err, "failed to ensure finalizer")
        return ctrl.Result{}, err
    }
    
    // 同步资源
    recLabels := map[string]string{
        "app.kubernetes.io/name": devbox.Name,
    }
    
    if err := r.syncSecret(ctx, devbox, recLabels); err != nil {
        logger.Error(err, "failed to sync secret")
        r.Recorder.Eventf(devbox, corev1.EventTypeWarning,
            "SyncSecretFailed", "Failed to sync secret: %v", err)
        return ctrl.Result{}, err
    }
    
    if err := r.syncNetwork(ctx, devbox, recLabels); err != nil {
        logger.Error(err, "failed to sync network")
        r.Recorder.Eventf(devbox, corev1.EventTypeWarning,
            "SyncNetworkFailed", "Failed to sync network: %v", err)
        return ctrl.Result{}, err
    }
    
    if err := r.syncPod(ctx, devbox, recLabels); err != nil {
        logger.Error(err, "failed to sync pod")
        r.Recorder.Eventf(devbox, corev1.EventTypeWarning,
            "SyncPodFailed", "Failed to sync pod: %v", err)
        return ctrl.Result{}, err
    }
    
    if err := r.syncDevboxPhase(ctx, devbox, recLabels); err != nil {
        logger.Error(err, "failed to sync phase")
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

---

## 7. 最佳实践

### 7.1 ✅ 使用 client.IgnoreNotFound

```go
// ✅ 正确：忽略 NotFound 错误
if err := r.Get(ctx, key, obj); err != nil {
    return ctrl.Result{}, client.IgnoreNotFound(err)
}

// ❌ 错误：手动判断
if err := r.Get(ctx, key, obj); err != nil {
    if apierrors.IsNotFound(err) {
        return ctrl.Result{}, nil
    }
    return ctrl.Result{}, err
}
```

### 7.2 ✅ 错误包装

```go
// ✅ 正确：使用 %w 包装错误
return fmt.Errorf("failed to sync pod: %w", err)

// ❌ 错误：使用 %v 丢失错误链
return fmt.Errorf("failed to sync pod: %v", err)
```

### 7.3 ✅ 记录完整的错误上下文

```go
// ✅ 正确：记录详细上下文
logger.Error(err, "failed to sync pod",
    "devbox", devbox.Name,
    "namespace", devbox.Namespace,
    "podName", podName,
    "phase", devbox.Status.Phase)

// ❌ 错误：缺少上下文
logger.Error(err, "failed to sync pod")
```

### 7.4 ✅ 区分错误类型

```go
// ✅ 正确：根据错误类型决定处理方式
if err := r.syncPod(ctx, devbox); err != nil {
    if apierrors.IsInvalid(err) {
        // 验证错误：不重试
        logger.Error(err, "invalid pod spec")
        return ctrl.Result{}, nil
    }
    
    if apierrors.IsTooManyRequests(err) {
        // 限流：延迟重试
        return ctrl.Result{RequeueAfter: 1 * time.Minute}, nil
    }
    
    // 其他错误：立即重试
    return ctrl.Result{}, err
}
```

### 7.5 ✅ 更新 Status 失败的处理

```go
// ✅ 正确：Status 更新失败不影响主流程
if err := r.syncPod(ctx, devbox); err != nil {
    logger.Error(err, "failed to sync pod")
    
    // 尝试更新 Status，但不影响主流程
    devbox.Status.Phase = devboxv1alpha2.DevboxPhaseError
    if updateErr := r.Status().Update(ctx, devbox); updateErr != nil {
        logger.Error(updateErr, "failed to update status")
        // 不返回 updateErr，返回原始错误
    }
    
    return ctrl.Result{}, err  // 返回原始错误
}

// ❌ 错误：Status 更新失败覆盖原始错误
if err := r.syncPod(ctx, devbox); err != nil {
    devbox.Status.Phase = devboxv1alpha2.DevboxPhaseError
    if updateErr := r.Status().Update(ctx, devbox); updateErr != nil {
        return ctrl.Result{}, updateErr  // ❌ 丢失原始错误
    }
    return ctrl.Result{}, err
}
```

---

## 8. 错误恢复

### 8.1 自动恢复

```go
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 检查是否处于错误状态
    if devbox.Status.Phase == devboxv1alpha2.DevboxPhaseError {
        logger.Info("devbox in error state, attempting recovery", "devbox", devbox.Name)
        
        // 尝试恢复
        if err := r.recoverDevbox(ctx, devbox); err != nil {
            logger.Error(err, "recovery failed")
            return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil  // 延迟重试
        }
        
        // 恢复成功，清除错误状态
        devbox.Status.Phase = devboxv1alpha2.DevboxPhasePending
        r.Status().Update(ctx, devbox)
        
        logger.Info("devbox recovered", "devbox", devbox.Name)
        r.Recorder.Event(devbox, corev1.EventTypeNormal, "Recovered", "Devbox recovered from error state")
    }
    
    // 正常处理
    return r.reconcile(ctx, devbox)
}
```

### 8.2 重试计数

```go
type DevboxStatus struct {
    Phase        Phase  `json:"phase,omitempty"`
    RetryCount   int    `json:"retryCount,omitempty"`
    LastError    string `json:"lastError,omitempty"`
}

func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if err := r.syncPod(ctx, devbox); err != nil {
        // 增加重试计数
        devbox.Status.RetryCount++
        devbox.Status.LastError = err.Error()
        
        // 超过最大重试次数，放弃
        if devbox.Status.RetryCount > 10 {
            logger.Error(err, "max retries exceeded", "devbox", devbox.Name)
            devbox.Status.Phase = devboxv1alpha2.DevboxPhaseError
            r.Status().Update(ctx, devbox)
            return ctrl.Result{}, nil  // 不再重试
        }
        
        r.Status().Update(ctx, devbox)
        return ctrl.Result{RequeueAfter: time.Duration(devbox.Status.RetryCount) * 30 * time.Second}, nil
    }
    
    // 成功，重置计数
    if devbox.Status.RetryCount > 0 {
        devbox.Status.RetryCount = 0
        devbox.Status.LastError = ""
        r.Status().Update(ctx, devbox)
    }
    
    return ctrl.Result{}, nil
}
```

---

## 9. 总结

### 核心要点

1. ✅ **使用 IgnoreNotFound** - 忽略资源不存在错误
2. ✅ **区分错误类型** - 临时 vs 永久
3. ✅ **记录详细日志** - 结构化日志
4. ✅ **更新 Status** - 反馈错误给用户
5. ✅ **记录 Event** - 审计和通知
6. ✅ **合理的重试策略** - 避免无限重试

### 错误处理决策树

```
发生错误
  │
  ▼
NotFound?
  ├─ 是 → IgnoreNotFound
  └─ 否 → 继续
         │
         ▼
    临时错误?
      ├─ 是 → RequeueAfter 30s
      └─ 否 → 继续
             │
             ▼
        永久错误?
          ├─ 是 → 记录日志，不重试
          └─ 否 → 返回错误，立即重试
```

### 与其他模式的配合

- **RetryOnConflict**: 处理 Conflict 错误
- **Event Recording**: 记录错误事件
- **Request-Driven**: 在 Request Status 中反馈错误

### 参考资源

- [Kubernetes API Errors](https://pkg.go.dev/k8s.io/apimachinery/pkg/api/errors)
- [Controller Runtime Client](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client)
- [Sealos Devbox Controller](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
