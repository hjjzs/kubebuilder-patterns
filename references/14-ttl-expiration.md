# TTL/Expiration 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 会话管理、临时资源、自动清理

## 概述

TTL/Expiration 模式通过在资源的 Annotation 中记录时间戳，并在 Controller 中检查是否过期，实现资源的自动清理。常用于临时会话、缓存、临时令牌等场景。

## 快速开始

```go
// Terminal 资源的 TTL 管理
const KeepaliveAnnotation = "lastUpdateTime"

// 检查是否过期
func isExpired(terminal *Terminal) bool {
    anno := terminal.Annotations
    lastUpdateTime, err := time.Parse(time.RFC3339, anno[KeepaliveAnnotation])
    if err != nil {
        return false
    }
    
    duration, _ := time.ParseDuration(terminal.Spec.Keepalived)
    return lastUpdateTime.Add(duration).Before(time.Now())
}

// Reconcile 中处理过期
func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 检查是否过期
    if isExpired(terminal) {
        if err := r.Delete(ctx, terminal); err != nil {
            return ctrl.Result{}, client.IgnoreNotFound(err)
        }
        logger.Info("deleted expired terminal")
        return ctrl.Result{}, nil
    }
    
    // 正常处理
    // ...
    
    // 重新入队，在过期前再次检查
    duration, _ := time.ParseDuration(terminal.Spec.Keepalived)
    return ctrl.Result{RequeueAfter: duration}, nil
}
```

---

## 1. 实现方式

### 1.1 基于 Annotation 的 TTL

```go
// controllers/terminal/controllers/terminal_controller.go
const KeepaliveAnnotation = "lastUpdateTime"

// CRD 定义
type TerminalSpec struct {
    // Keepalived is the duration to keep the terminal alive
    // +kubebuilder:default:="24h"
    Keepalived string `json:"keepalived,omitempty"`
}

// 初始化 keepalive 时间戳
func (r *TerminalReconciler) fillDefaultValue(ctx context.Context, terminal *Terminal) error {
    if terminal.Annotations == nil {
        terminal.Annotations = make(map[string]string)
    }
    
    if _, ok := terminal.Annotations[KeepaliveAnnotation]; !ok {
        terminal.Annotations[KeepaliveAnnotation] = time.Now().Format(time.RFC3339)
        return r.Update(ctx, terminal)
    }
    
    return nil
}

// 更新 keepalive 时间戳（通过 API 调用）
func (r *TerminalReconciler) UpdateKeepalive(ctx context.Context, name, namespace string) error {
    terminal := &Terminal{}
    if err := r.Get(ctx, client.ObjectKey{Name: name, Namespace: namespace}, terminal); err != nil {
        return err
    }
    
    terminal.Annotations[KeepaliveAnnotation] = time.Now().Format(time.RFC3339)
    return r.Update(ctx, terminal)
}

// 检查是否过期
func isExpired(terminal *Terminal) bool {
    anno := terminal.Annotations
    lastUpdateTime, err := time.Parse(time.RFC3339, anno[KeepaliveAnnotation])
    if err != nil {
        return false  // 解析失败，不认为过期
    }
    
    duration, err := time.ParseDuration(terminal.Spec.Keepalived)
    if err != nil {
        return false  // 解析失败，不认为过期
    }
    
    return lastUpdateTime.Add(duration).Before(time.Now())
}
```

### 1.2 基于 Status 的 TTL

```go
type TerminalStatus struct {
    // LastUpdateTime is the last time the terminal was updated
    LastUpdateTime *metav1.Time `json:"lastUpdateTime,omitempty"`
}

func isExpired(terminal *Terminal) bool {
    if terminal.Status.LastUpdateTime == nil {
        return false
    }
    
    duration, _ := time.ParseDuration(terminal.Spec.Keepalived)
    expirationTime := terminal.Status.LastUpdateTime.Add(duration)
    
    return time.Now().After(expirationTime)
}
```

### 1.3 基于 CreationTimestamp 的 TTL

```go
// 简单场景：创建后固定时间过期
func isExpired(obj client.Object, ttl time.Duration) bool {
    expirationTime := obj.GetCreationTimestamp().Add(ttl)
    return time.Now().After(expirationTime)
}

// 使用
if isExpired(terminal, 24*time.Hour) {
    r.Delete(ctx, terminal)
}
```

---

## 2. Reconcile 实现

### 2.1 完整的 TTL 处理流程

```go
// controllers/terminal/controllers/terminal_controller.go
func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    terminal := &terminalv1.Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 1. 处理删除（Finalizer）
    if !terminal.DeletionTimestamp.IsZero() {
        if controllerutil.RemoveFinalizer(terminal, FinalizerName) {
            if err := r.Update(ctx, terminal); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }
    
    // 2. 添加 Finalizer
    if controllerutil.AddFinalizer(terminal, FinalizerName) {
        if err := r.Update(ctx, terminal); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 3. 初始化 keepalive 时间戳
    if err := r.fillDefaultValue(ctx, terminal); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. 检查是否过期
    if isExpired(terminal) {
        if err := r.Delete(ctx, terminal); err != nil {
            return ctrl.Result{}, client.IgnoreNotFound(err)
        }
        logger.Info("deleted expired terminal")
        return ctrl.Result{}, nil
    }
    
    // 5. 正常处理
    if err := r.syncResources(ctx, terminal); err != nil {
        return ctrl.Result{}, err
    }
    
    // 6. 重新入队，在过期前检查
    duration, _ := time.ParseDuration(terminal.Spec.Keepalived)
    return ctrl.Result{RequeueAfter: duration}, nil
}
```

### 2.2 优化：提前检查

```go
// ✅ 优化：在过期前 1 分钟检查，而不是等到过期
func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if isExpired(terminal) {
        r.Delete(ctx, terminal)
        return ctrl.Result{}, nil
    }
    
    // 计算下次检查时间
    duration, _ := time.ParseDuration(terminal.Spec.Keepalived)
    lastUpdateTime, _ := time.Parse(time.RFC3339, terminal.Annotations[KeepaliveAnnotation])
    expirationTime := lastUpdateTime.Add(duration)
    
    // 在过期前 1 分钟检查
    nextCheckTime := expirationTime.Add(-1 * time.Minute)
    requeueAfter := time.Until(nextCheckTime)
    
    if requeueAfter < 0 {
        requeueAfter = 0
    }
    
    return ctrl.Result{RequeueAfter: requeueAfter}, nil
}
```

---

## 3. Keepalive 机制

### 3.1 客户端更新 Keepalive

```go
// API 端点：更新 Terminal 的 keepalive
func (h *Handler) UpdateTerminalKeepalive(w http.ResponseWriter, r *http.Request) {
    terminalName := r.URL.Query().Get("name")
    namespace := r.URL.Query().Get("namespace")
    
    terminal := &terminalv1.Terminal{}
    if err := h.Client.Get(r.Context(), client.ObjectKey{
        Name:      terminalName,
        Namespace: namespace,
    }, terminal); err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }
    
    // 更新 keepalive 时间戳
    terminal.Annotations[KeepaliveAnnotation] = time.Now().Format(time.RFC3339)
    if err := h.Client.Update(r.Context(), terminal); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusOK)
}
```

### 3.2 前端定时发送 Keepalive

```javascript
// 前端代码：每 5 分钟发送一次 keepalive
const KEEPALIVE_INTERVAL = 5 * 60 * 1000; // 5 分钟

function startKeepalive(terminalName, namespace) {
    const interval = setInterval(async () => {
        try {
            await fetch(`/api/terminal/keepalive?name=${terminalName}&namespace=${namespace}`, {
                method: 'POST',
            });
            console.log('Keepalive sent');
        } catch (err) {
            console.error('Failed to send keepalive:', err);
        }
    }, KEEPALIVE_INTERVAL);
    
    // 页面关闭时清理
    window.addEventListener('beforeunload', () => {
        clearInterval(interval);
    });
}
```

---

## 4. 最佳实践

### 4.1 ✅ 使用 RFC3339 格式

```go
// ✅ 正确：使用标准时间格式
terminal.Annotations[KeepaliveAnnotation] = time.Now().Format(time.RFC3339)
// 输出: "2026-01-23T10:30:00Z"

// ❌ 错误：使用自定义格式
terminal.Annotations[KeepaliveAnnotation] = time.Now().Format("2006-01-02 15:04:05")
// 难以解析，不同时区有问题
```

### 4.2 ✅ 处理解析错误

```go
// ✅ 正确：处理解析错误
func isExpired(terminal *Terminal) bool {
    lastUpdateTime, err := time.Parse(time.RFC3339, terminal.Annotations[KeepaliveAnnotation])
    if err != nil {
        // 解析失败，不认为过期（避免误删除）
        return false
    }
    
    duration, err := time.ParseDuration(terminal.Spec.Keepalived)
    if err != nil {
        return false
    }
    
    return lastUpdateTime.Add(duration).Before(time.Now())
}
```

### 4.3 ✅ 合理的 RequeueAfter

```go
// ✅ 正确：在过期前重新检查
duration, _ := time.ParseDuration(terminal.Spec.Keepalived)
return ctrl.Result{RequeueAfter: duration}, nil

// ❌ 错误：固定间隔检查（浪费资源）
return ctrl.Result{RequeueAfter: 1 * time.Minute}, nil
```

### 4.4 ✅ 记录删除日志

```go
if isExpired(terminal) {
    logger.Info("deleting expired terminal",
        "terminal", terminal.Name,
        "lastUpdateTime", terminal.Annotations[KeepaliveAnnotation],
        "keepalived", terminal.Spec.Keepalived,
    )
    
    if err := r.Delete(ctx, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    r.Recorder.Event(terminal, corev1.EventTypeNormal, 
        "Expired", "Terminal expired and deleted")
    
    return ctrl.Result{}, nil
}
```

---

## 5. 实际案例

### 案例 1：Terminal 控制器的会话管理

```go
// controllers/terminal/controllers/terminal_controller.go
type TerminalSpec struct {
    // Keepalived is the duration to keep the terminal alive
    // +kubebuilder:default:="24h"
    Keepalived string `json:"keepalived,omitempty"`
}

func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx, "terminal", req.NamespacedName)
    
    terminal := &terminalv1.Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // Finalizer 处理
    if !terminal.DeletionTimestamp.IsZero() {
        if controllerutil.RemoveFinalizer(terminal, FinalizerName) {
            if err := r.Update(ctx, terminal); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }
    
    if controllerutil.AddFinalizer(terminal, FinalizerName) {
        if err := r.Update(ctx, terminal); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 初始化 keepalive
    if err := r.fillDefaultValue(ctx, terminal); err != nil {
        return ctrl.Result{}, err
    }
    
    // 检查过期
    if isExpired(terminal) {
        if err := r.Delete(ctx, terminal); err != nil {
            return ctrl.Result{}, err
        }
        logger.Info("delete expired terminal success")
        return ctrl.Result{}, nil
    }
    
    // 同步资源
    recLabels := label.RecommendedLabels(&label.Recommended{
        Name:      terminal.Name,
        ManagedBy: label.DefaultManagedBy,
        PartOf:    "terminal",
    })
    
    var hostname string
    if err := r.syncDeployment(ctx, terminal, &hostname, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    
    if err := r.syncService(ctx, terminal, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    
    if err := r.syncIngress(ctx, terminal, hostname); err != nil {
        return ctrl.Result{}, err
    }
    
    // 重新入队
    duration, _ := time.ParseDuration(terminal.Spec.Keepalived)
    return ctrl.Result{RequeueAfter: duration}, nil
}
```

---

## 6. 高级用法

### 6.1 多级 TTL

```go
// 不同类型的资源有不同的 TTL
type ResourceTTL struct {
    ShortLived  time.Duration  // 1 小时
    MediumLived time.Duration  // 24 小时
    LongLived   time.Duration  // 7 天
}

func (r *TerminalReconciler) getTTL(terminal *Terminal) time.Duration {
    switch terminal.Spec.Type {
    case "temporary":
        return 1 * time.Hour
    case "development":
        return 24 * time.Hour
    case "production":
        return 7 * 24 * time.Hour
    default:
        return 24 * time.Hour
    }
}
```

### 6.2 优雅过期（提前通知）

```go
func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    duration, _ := time.ParseDuration(terminal.Spec.Keepalived)
    lastUpdateTime, _ := time.Parse(time.RFC3339, terminal.Annotations[KeepaliveAnnotation])
    expirationTime := lastUpdateTime.Add(duration)
    timeUntilExpiration := time.Until(expirationTime)
    
    // 提前 10 分钟发送通知
    if timeUntilExpiration < 10*time.Minute && timeUntilExpiration > 0 {
        if terminal.Annotations["expiration-warning-sent"] != "true" {
            r.Recorder.Eventf(terminal, corev1.EventTypeWarning,
                "ExpirationWarning",
                "Terminal will expire in %v", timeUntilExpiration)
            
            terminal.Annotations["expiration-warning-sent"] = "true"
            r.Update(ctx, terminal)
        }
    }
    
    // 过期删除
    if timeUntilExpiration <= 0 {
        r.Delete(ctx, terminal)
        return ctrl.Result{}, nil
    }
    
    // 重新入队
    return ctrl.Result{RequeueAfter: timeUntilExpiration}, nil
}
```

---

## 7. 总结

### 核心要点

1. ✅ **使用 Annotation 存储时间戳** - 或使用 Status 字段
2. ✅ **使用 RFC3339 格式** - 标准时间格式
3. ✅ **处理解析错误** - 避免误删除
4. ✅ **合理的 RequeueAfter** - 在过期前检查
5. ✅ **记录删除日志** - 便于审计

### 适用场景

| 场景 | 是否使用 TTL |
|------|------------|
| 临时会话 | ✅ 是 |
| 临时令牌 | ✅ 是 |
| 缓存资源 | ✅ 是 |
| 持久化资源 | ❌ 否 |

### 与其他模式的配合

- **Finalizer**: 删除前清理子资源
- **Event Recording**: 记录过期删除事件
- **Request-Driven**: Request 完成后自动清理

### 参考资源

- [Sealos Terminal Controller](https://github.com/labring/sealos/blob/main/controllers/terminal/controllers/terminal_controller.go)
- [TTL Controller](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
