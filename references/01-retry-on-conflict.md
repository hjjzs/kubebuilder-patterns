# RetryOnConflict 模式

**复杂度**: 🟢 简单  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 资源状态更新、Finalizer 管理

## 概述

RetryOnConflict 是 Kubernetes client-go 提供的乐观锁冲突重试机制。当多个客户端或控制器同时修改同一资源时，通过自动重试解决并发冲突问题。

## 目录

1. [问题背景](#1-问题背景)
2. [工作原理](#2-工作原理)
3. [使用方法](#3-使用方法)
4. [优缺点分析](#4-优缺点分析)
5. [最佳实践](#5-最佳实践)
6. [常见错误](#6-常见错误)
7. [实际案例](#7-实际案例)

---

## 1. 问题背景

### 1.1 Kubernetes 的乐观锁机制

Kubernetes 使用 `resourceVersion` 字段实现乐观并发控制：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example
  resourceVersion: "12345"  # 每次更新后递增
data:
  key: value
```

**更新流程**：

```
1. Client A: GET resource (resourceVersion: 12345)
2. Client B: GET resource (resourceVersion: 12345)
3. Client A: UPDATE with resourceVersion 12345 → 成功 (新版本: 12346)
4. Client B: UPDATE with resourceVersion 12345 → 失败 (409 Conflict)
```

### 1.2 冲突场景示例

在 Sealos Devbox 控制器中，多个 Reconcile 循环可能同时更新同一 Devbox 的状态：

```
时间线:
T1: Reconcile-1 读取 Devbox (resourceVersion: 100)
T2: Reconcile-2 读取 Devbox (resourceVersion: 100)
T3: Reconcile-1 更新 Phase=Running → 成功 (resourceVersion: 101)
T4: Reconcile-2 更新 ContentID=abc → 失败 (409 Conflict)
```

**没有重试机制的后果**：
- ❌ 状态更新丢失
- ❌ 数据不一致
- ❌ 需要等待下次 Reconcile（可能很久）

---

## 2. 工作原理

### 2.1 函数签名

```go
// k8s.io/client-go/util/retry
func RetryOnConflict(backoff wait.Backoff, fn func() error) error
```

**参数说明**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `backoff` | `wait.Backoff` | 重试策略配置 |
| `fn` | `func() error` | 要执行的操作（闭包） |

### 2.2 预定义的 Backoff 策略

```go
// DefaultRetry: 推荐用于多客户端修改同一资源
var DefaultRetry = wait.Backoff{
    Steps:    5,           // 最多重试 5 次
    Duration: 10 * time.Millisecond,
    Factor:   1.0,         // 固定延迟
    Jitter:   0.1,         // 10% 随机抖动
}

// DefaultBackoff: 推荐用于控制器主动管理的资源
var DefaultBackoff = wait.Backoff{
    Steps:    4,           // 最多重试 4 次
    Duration: 10 * time.Millisecond,
    Factor:   5.0,         // 指数增长
    Jitter:   0.1,
}
```

**延迟计算**：

```
DefaultRetry:
  Attempt 1: 10ms ± 1ms
  Attempt 2: 10ms ± 1ms
  Attempt 3: 10ms ± 1ms
  ...

DefaultBackoff:
  Attempt 1: 10ms ± 1ms
  Attempt 2: 50ms ± 5ms
  Attempt 3: 250ms ± 25ms
  Attempt 4: 1250ms ± 125ms
```

### 2.3 执行流程

```
┌─────────────────────────────────────────────────────────────┐
│ RetryOnConflict(retry.DefaultRetry, func() error {         │
│   1. 获取最新资源 (GET)                                      │
│   2. 修改资源字段                                            │
│   3. 提交更新 (UPDATE)                                       │
│   return err                                                │
│ })                                                          │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │ 执行 fn()        │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ err == nil?      │
              └────────┬─────────┘
                       │
           ┌───────────┴───────────┐
           │ Yes                   │ No
           ▼                       ▼
    ┌────────────┐      ┌──────────────────┐
    │ 返回成功   │      │ 是 Conflict 错误? │
    └────────────┘      └────────┬─────────┘
                                 │
                     ┌───────────┴───────────┐
                     │ Yes                   │ No
                     ▼                       ▼
          ┌──────────────────┐      ┌──────────────┐
          │ 重试次数用完?    │      │ 返回错误     │
          └────────┬─────────┘      └──────────────┘
                   │
       ┌───────────┴───────────┐
       │ Yes                   │ No
       ▼                       ▼
┌────────────┐      ┌──────────────────┐
│ 返回错误   │      │ 等待 backoff     │
└────────────┘      │ 后重新执行 fn()  │
                    └──────────────────┘
```

---

## 3. 使用方法

### 3.1 基本用法：更新状态

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) updateDevboxStatus(
    ctx context.Context, 
    devbox *devboxv1alpha2.Devbox,
) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        // 1. 重新获取最新版本
        latestDevbox := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latestDevbox); err != nil {
            return err
        }
        
        // 2. 修改状态字段
        latestDevbox.Status.Phase = devbox.Status.Phase
        latestDevbox.Status.LastContainerStatus = devbox.Status.LastContainerStatus
        
        // 3. 提交更新
        return r.Status().Update(ctx, latestDevbox)
    })
}
```

**关键点**：
- ✅ 每次重试都重新 `Get` 最新资源
- ✅ 只修改需要更新的字段
- ✅ 使用 `Status().Update()` 更新状态子资源

### 3.2 添加 Finalizer

```go
// controllers/license/internal/controller/license_controller.go
func (r *LicenseReconciler) ensureFinalizer(
    ctx context.Context,
    req ctrl.Request,
) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &licensev1.License{}
        if err := r.Get(ctx, req.NamespacedName, latest); err != nil {
            return err
        }
        
        // AddFinalizer 返回 bool 表示是否有变更
        if controllerutil.AddFinalizer(latest, licenseFinalizer) {
            return r.Update(ctx, latest)
        }
        return nil  // 已存在，无需更新
    })
}
```

### 3.3 移除 Finalizer

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) removeFinalizer(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
            return client.IgnoreNotFound(err)  // 已删除，忽略
        }
        
        if controllerutil.RemoveFinalizer(latest, FinalizerName) {
            return r.Update(ctx, latest)
        }
        return nil
    })
}
```

### 3.4 更新多个字段

```go
func (r *DevboxReconciler) updateNetworkStatus(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    networkType string,
    nodePort int32,
) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
            return err
        }
        
        // 同时更新多个字段
        latest.Status.Network.Type = networkType
        latest.Status.Network.NodePort = nodePort
        
        return r.Status().Update(ctx, latest)
    })
}
```

---

## 4. 优缺点分析

### 4.1 优点

| 优点 | 说明 | 影响 |
|------|------|------|
| ✅ **自动处理冲突** | 无需手动编写重试逻辑 | 代码简洁 |
| ✅ **保证数据一致性** | 每次重试都获取最新版本 | 避免覆盖其他客户端的修改 |
| ✅ **指数退避** | 智能延迟策略 | 减少 API Server 压力 |
| ✅ **最终一致性** | 高并发下仍能成功 | 提高可靠性 |
| ✅ **防止数据丢失** | 不会覆盖其他字段的更新 | 多控制器协作安全 |

### 4.2 缺点

| 缺点 | 说明 | 缓解措施 |
|------|------|---------|
| ❌ **性能开销** | 每次重试需要额外的 GET + UPDATE | 使用 Predicate 减少 Reconcile 频率 |
| ❌ **可能失败** | 重试次数耗尽后仍会失败 | 返回错误触发 Requeue |
| ❌ **调试困难** | 重试过程隐式，日志不明显 | 添加详细日志 |
| ❌ **闭包复杂性** | 代码必须幂等 | 遵循最佳实践 |
| ❌ **延迟增加** | 重试会增加操作延迟 | 使用 DefaultRetry（固定延迟） |

### 4.3 性能影响分析

**场景：10 个 Reconcile 同时更新同一 Devbox**

| 指标 | 无重试 | 使用 RetryOnConflict |
|------|--------|---------------------|
| 成功更新数 | 1 | 10 |
| 失败更新数 | 9 | 0 |
| 平均延迟 | 50ms | 80ms (含重试) |
| API 调用次数 | 20 (10 GET + 10 UPDATE) | 35 (10 GET + 15 重试 GET + 10 UPDATE) |

**结论**：虽然 API 调用增加 75%，但避免了 9 次更新丢失。

---

## 5. 最佳实践

### 5.1 ✅ 始终在闭包内重新 Get

```go
// ✅ 正确：每次重试都获取最新版本
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)  // 在闭包内
    latest.Status.Phase = "Running"
    return r.Status().Update(ctx, latest)
})

// ❌ 错误：使用闭包外的过期对象
devbox := &Devbox{}
r.Get(ctx, key, devbox)
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    devbox.Status.Phase = "Running"  // resourceVersion 过期
    return r.Status().Update(ctx, devbox)
})
```

### 5.2 ✅ 确保闭包内代码幂等

```go
// ✅ 正确：幂等操作
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.Phase = "Running"  // 设置固定值，幂等
    return r.Status().Update(ctx, latest)
})

// ❌ 错误：非幂等操作
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.RetryCount++  // 每次重试都递增，非幂等
    return r.Status().Update(ctx, latest)
})
```

### 5.3 ✅ 避免在闭包内执行有副作用的操作

```go
// ❌ 错误：发送通知可能重复
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    
    // 如果重试 3 次，通知会发送 3 次
    r.sendNotification("Devbox updated")  // ❌
    
    latest.Status.Phase = "Running"
    return r.Status().Update(ctx, latest)
})

// ✅ 正确：副作用操作放在闭包外
err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.Phase = "Running"
    return r.Status().Update(ctx, latest)
})
if err == nil {
    r.sendNotification("Devbox updated")  // ✅ 只发送一次
}
```

### 5.4 ✅ 使用 IgnoreNotFound 处理删除场景

```go
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    if err := r.Get(ctx, key, latest); err != nil {
        return client.IgnoreNotFound(err)  // 资源已删除，不算错误
    }
    
    controllerutil.RemoveFinalizer(latest, FinalizerName)
    return r.Update(ctx, latest)
})
```

### 5.5 ✅ 选择合适的 Backoff 策略

```go
// 场景 1: 多个客户端同时修改（高冲突）
// 使用 DefaultRetry（固定延迟，更多重试）
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    // 更新用户权限（多个 API 调用可能同时修改）
})

// 场景 2: 控制器主动管理（低冲突）
// 使用 DefaultBackoff（指数退避，更少重试）
retry.RetryOnConflict(retry.DefaultBackoff, func() error {
    // 更新控制器管理的资源状态
})
```

---

## 6. 常见错误

### 6.1 ❌ 在闭包外缓存资源

**错误代码**：

```go
devbox := &Devbox{}
r.Get(ctx, key, devbox)  // resourceVersion: 100

retry.RetryOnConflict(retry.DefaultRetry, func() error {
    // 使用闭包外的 devbox，resourceVersion 永远是 100
    devbox.Status.Phase = "Running"
    return r.Status().Update(ctx, devbox)  // 永远失败
})
```

**问题**：每次重试都使用相同的 `resourceVersion: 100`，导致无限重试失败。

**修复**：

```go
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    devbox := &Devbox{}
    r.Get(ctx, key, devbox)  // 每次重试都获取最新版本
    devbox.Status.Phase = "Running"
    return r.Status().Update(ctx, devbox)
})
```

### 6.2 ❌ 在闭包内创建资源

**错误代码**：

```go
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    pod := &corev1.Pod{...}
    return r.Create(ctx, pod)  // 重试时会重复创建
})
```

**问题**：`Create` 操作非幂等，重试会导致 `AlreadyExists` 错误。

**修复**：

```go
// 方式 1: 不使用 RetryOnConflict
pod := &corev1.Pod{...}
if err := r.Create(ctx, pod); err != nil {
    if !apierrors.IsAlreadyExists(err) {
        return err
    }
}

// 方式 2: 使用 CreateOrUpdate
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, pod, func() error {
    // 设置期望状态
    return nil
})
```

### 6.3 ❌ 忽略非冲突错误

**错误代码**：

```go
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    if err := r.Get(ctx, key, latest); err != nil {
        return nil  // ❌ 忽略所有错误
    }
    latest.Status.Phase = "Running"
    return r.Status().Update(ctx, latest)
})
```

**问题**：`Get` 失败（如网络错误）时，返回 `nil` 导致静默失败。

**修复**：

```go
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    if err := r.Get(ctx, key, latest); err != nil {
        return err  // ✅ 返回错误，让 RetryOnConflict 判断是否重试
    }
    latest.Status.Phase = "Running"
    return r.Status().Update(ctx, latest)
})
```

### 6.4 ❌ 修改 Spec 字段

**错误代码**：

```go
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Spec.State = "Running"  // ❌ 控制器不应修改 Spec
    return r.Update(ctx, latest)
})
```

**问题**：Spec 是用户期望状态，控制器应该只修改 Status。

**修复**：

```go
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.Phase = "Running"  // ✅ 只修改 Status
    return r.Status().Update(ctx, latest)
})
```

---

## 7. 实际案例

### 7.1 案例 1：Devbox 容器状态更新

**场景**：Pod 状态变化时，更新 Devbox 的 `LastContainerStatus`。

```go
// controllers/devbox/internal/controller/devbox_controller.go:918-932
func (r *DevboxReconciler) updateContainerStatus(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    pod *corev1.Pod,
) error {
    if len(pod.Status.ContainerStatuses) == 0 {
        return nil
    }
    
    err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latestDevbox := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latestDevbox); err != nil {
            return err
        }
        latestDevbox.Status.LastContainerStatus = pod.Status.ContainerStatuses[0]
        return r.Status().Update(ctx, latestDevbox)
    })
    
    if err != nil {
        logger.Error(err, "failed to update devbox status")
        return err
    }
    return nil
}
```

**分析**：
- ✅ 每次重试都重新 Get Devbox
- ✅ 只更新 `LastContainerStatus` 字段
- ✅ 使用 `Status().Update()` 更新状态子资源
- ✅ 记录错误日志便于调试

### 7.2 案例 2：License Finalizer 管理

**场景**：确保 License 资源有 Finalizer，防止直接删除。

```go
// controllers/license/internal/controller/license_controller.go:88-99
func (r *LicenseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    license := &licensev1.License{}
    if err := r.Get(ctx, req.NamespacedName, license); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if license.DeletionTimestamp.IsZero() {
        addedFinalizer := false
        if err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
            latest := &licensev1.License{}
            if err := r.Get(ctx, req.NamespacedName, latest); err != nil {
                return err
            }
            changed := controllerutil.AddFinalizer(latest, licenseFinalizer)
            if !changed {
                return nil  // 已存在，无需更新
            }
            addedFinalizer = true
            return r.Update(ctx, latest)
        }); err != nil {
            return ctrl.Result{}, err
        }
        
        if addedFinalizer {
            return ctrl.Result{Requeue: true}, nil  // 重新入队，等待 Finalizer 生效
        }
    }
    // ... 继续处理
}
```

**分析**：
- ✅ 使用 `AddFinalizer` 的返回值判断是否需要更新
- ✅ 添加 Finalizer 后立即 Requeue，确保后续逻辑看到最新状态
- ✅ 避免不必要的 Update 调用

### 7.3 案例 3：User 控制器的 Namespace 创建

**场景**：为用户创建 Namespace，并设置 Owner 标签。

```go
// controllers/user/controllers/user_controller.go:244-250
func (r *UserReconciler) createNamespace(
    ctx context.Context,
    user *userv1.User,
) error {
    if err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        ns := &v1.Namespace{}
        ns.Name = config.GetUsersNamespace(user.Name)
        ns.Labels = map[string]string{
            userv1.UserLabelOwnerKey: user.Name,
        }
        
        _, err := controllerutil.CreateOrUpdate(ctx, r.Client, ns, func() error {
            // 确保标签正确
            if ns.Labels == nil {
                ns.Labels = make(map[string]string)
            }
            ns.Labels[userv1.UserLabelOwnerKey] = user.Name
            return nil
        })
        return err
    }); err != nil {
        return fmt.Errorf("failed to create namespace: %w", err)
    }
    return nil
}
```

**分析**：
- ✅ 结合 `CreateOrUpdate` 实现幂等创建
- ✅ 在 Mutate 函数中确保标签正确
- ✅ 外层使用 `RetryOnConflict` 处理并发冲突

### 7.4 案例 4：处理删除场景

**场景**：移除 Pod 的 Finalizer 并删除，需要处理 Pod 已被删除的情况。

```go
// controllers/devbox/internal/controller/devbox_controller.go:935-950
func (r *DevboxReconciler) deletePod(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    pod *corev1.Pod,
) error {
    originalPodUID := pod.UID
    
    // 移除 Finalizer
    err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latestPod := &corev1.Pod{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(pod), latestPod); err != nil {
            if apierrors.IsNotFound(err) {
                return nil  // Pod 已删除，成功
            }
            return err
        }
        
        // UID 检查：防止操作重建的 Pod
        if latestPod.UID != originalPodUID {
            logger.Info("pod UID changed, skip finalizer removal")
            return nil
        }
        
        controllerutil.RemoveFinalizer(latestPod, FinalizerName)
        return r.Update(ctx, latestPod)
    })
    
    if err != nil {
        return err
    }
    
    // 删除 Pod
    return r.Delete(ctx, pod,
        client.GracePeriodSeconds(0),
        client.PropagationPolicy(metav1.DeletePropagationBackground),
    )
}
```

**分析**：
- ✅ 使用 `IsNotFound` 判断资源已删除
- ✅ UID 检查防止操作重建的资源
- ✅ 先移除 Finalizer，再删除资源

---

## 8. 性能优化建议

### 8.1 减少冲突频率

**使用 Predicate 过滤不必要的 Reconcile**：

```go
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}, builder.WithPredicates(
            predicate.Or(
                predicate.GenerationChangedPredicate{},  // 只在 Spec 变化时触发
                ContentIDChangedPredicate{},             // 只在 ContentID 变化时触发
            ),
        )).
        Complete(r)
}
```

### 8.2 批量更新

**合并多个字段的更新**：

```go
// ❌ 低效：多次更新
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.Phase = "Running"
    return r.Status().Update(ctx, latest)
})

retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.Network.Type = "NodePort"
    return r.Status().Update(ctx, latest)
})

// ✅ 高效：一次更新
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.Phase = "Running"
    latest.Status.Network.Type = "NodePort"
    return r.Status().Update(ctx, latest)
})
```

### 8.3 添加监控指标

```go
func (r *DevboxReconciler) updateStatusWithMetrics(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
) error {
    startTime := time.Now()
    retryCount := 0
    
    err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        retryCount++
        latest := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
            return err
        }
        latest.Status.Phase = devbox.Status.Phase
        return r.Status().Update(ctx, latest)
    })
    
    // 记录指标
    duration := time.Since(startTime)
    metrics.RecordStatusUpdate(duration, retryCount, err == nil)
    
    return err
}
```

---

## 9. 调试技巧

### 9.1 添加详细日志

```go
retry.RetryOnConflict(retry.DefaultRetry, func() error {
    logger.V(1).Info("attempting to update devbox status")
    
    latest := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
        logger.Error(err, "failed to get latest devbox")
        return err
    }
    
    logger.V(1).Info("updating devbox status",
        "oldPhase", latest.Status.Phase,
        "newPhase", devbox.Status.Phase,
        "resourceVersion", latest.ResourceVersion,
    )
    
    latest.Status.Phase = devbox.Status.Phase
    if err := r.Status().Update(ctx, latest); err != nil {
        logger.Error(err, "failed to update devbox status",
            "resourceVersion", latest.ResourceVersion,
        )
        return err
    }
    
    logger.V(1).Info("successfully updated devbox status")
    return nil
})
```

### 9.2 检测重试次数

```go
func RetryOnConflictWithCount(backoff wait.Backoff, fn func(attempt int) error) (int, error) {
    attempt := 0
    err := retry.RetryOnConflict(backoff, func() error {
        attempt++
        return fn(attempt)
    })
    return attempt, err
}

// 使用
attempts, err := RetryOnConflictWithCount(retry.DefaultRetry, func(attempt int) error {
    logger.Info("update attempt", "attempt", attempt)
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.Phase = "Running"
    return r.Status().Update(ctx, latest)
})

if err != nil {
    logger.Error(err, "update failed after retries", "attempts", attempts)
}
```

---

## 10. 总结

### 核心要点

1. ✅ **始终在闭包内重新 Get** - 确保使用最新的 resourceVersion
2. ✅ **确保闭包幂等** - 重试不应产生副作用
3. ✅ **选择合适的 Backoff** - 根据冲突频率选择策略
4. ✅ **处理 NotFound** - 资源删除时优雅处理
5. ✅ **避免在闭包内创建资源** - Create 操作非幂等

### 适用场景

| 场景 | 是否使用 RetryOnConflict |
|------|------------------------|
| 更新资源状态 | ✅ 是 |
| 添加/移除 Finalizer | ✅ 是 |
| 更新资源标签/注解 | ✅ 是 |
| 创建新资源 | ❌ 否（使用 CreateOrUpdate） |
| 删除资源 | ❌ 否（Delete 是幂等的） |
| 读取资源 | ❌ 否（Get 不会冲突） |

### 与其他模式的配合

- **CreateOrUpdate 模式**：在 CreateOrUpdate 的 Mutate 函数外使用 RetryOnConflict
- **Finalizer 模式**：添加/移除 Finalizer 时必须使用
- **Pipeline 模式**：在 Pipeline 的最后一步批量更新状态

### 参考资源

- [client-go retry 包文档](https://pkg.go.dev/k8s.io/client-go/util/retry)
- [Kubernetes API Conventions - Concurrency Control](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#concurrency-control-and-consistency)
- [Sealos Devbox Controller 实现](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
