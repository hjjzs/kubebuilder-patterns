# Concurrent Reconcile 模式

**复杂度**: 🟢 简单  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 大量资源、提升吞吐量、并发控制

## 概述

Concurrent Reconcile 模式通过配置 `MaxConcurrentReconciles` 参数，允许控制器并发处理多个 Reconcile 请求，显著提升处理大量资源时的吞吐量。

## 快速开始

### 默认配置 vs 并发配置

```go
// ❌ 默认：串行处理，一次只处理一个资源
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}

// ✅ 并发：同时处理 10 个资源
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 10,
        }).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}
```

---

## 1. 工作原理

### 1.1 串行 vs 并发

```
串行处理（默认 MaxConcurrentReconciles: 1）
┌─────────────────────────────────────────────────────┐
│ Queue: [Devbox-1, Devbox-2, Devbox-3, Devbox-4, ...] │
└─────────────────────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │ Worker 1              │
        │ Processing Devbox-1   │
        └───────────────────────┘
        
处理时间：N 个资源 × 平均处理时间


并发处理（MaxConcurrentReconciles: 10）
┌─────────────────────────────────────────────────────┐
│ Queue: [Devbox-1, Devbox-2, Devbox-3, Devbox-4, ...] │
└─────────────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Worker 1    │ │ Worker 2    │ │ Worker 10   │
│ Devbox-1    │ │ Devbox-2    │ │ Devbox-10   │
└─────────────┘ └─────────────┘ └─────────────┘

处理时间：N 个资源 × 平均处理时间 / 10
```

---

## 2. 使用方法

### 2.1 基本配置

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 10,  // 并发数
        }).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}
```

### 2.2 动态并发数

```go
// controllers/user/controllers/user_controller.go
import "github.com/labring/sealos/controllers/user/controllers/helper/ratelimiter"

func (r *UserReconciler) SetupWithManager(mgr ctrl.Manager, opts ratelimiter.Options) error {
    return ctrl.NewControllerManagedBy(mgr).
        WithOptions(kubecontroller.Options{
            MaxConcurrentReconciles: ratelimiter.GetConcurrent(opts),  // 从配置读取
            RateLimiter:             ratelimiter.GetRateLimiter(opts),
        }).
        For(&userv1.User{}).
        Complete(r)
}

// 从环境变量读取
func GetConcurrent(opts Options) int {
    if opts.Concurrent > 0 {
        return opts.Concurrent
    }
    // 默认值
    return 3
}
```

### 2.3 结合 RateLimiter

```go
// controllers/account/main.go
import "k8s.io/client-go/util/workqueue"

func main() {
    // 从环境变量读取配置
    concurrent := env.GetIntEnvWithDefault("CONCURRENT_RECONCILES", 10)
    
    rateLimiterOptions := workqueue.RateLimiterOptions{
        BaseDelay:         time.Second,
        MaxDelay:          1000 * time.Second,
        FailureMaxDelay:   1000 * time.Second,
    }
    
    rateOpts := controller.Options{
        MaxConcurrentReconciles: concurrent,
        RateLimiter:             workqueue.NewItemExponentialFailureRateLimiter(
            rateLimiterOptions.BaseDelay,
            rateLimiterOptions.MaxDelay,
        ),
    }
    
    if err = (&controllers.AccountReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr, rateOpts); err != nil {
        setupLog.Error(err, "unable to create controller")
        os.Exit(1)
    }
}
```

---

## 3. 性能对比

### 3.1 吞吐量提升

**场景**: 1000 个 Devbox 资源需要处理，每个耗时 1 秒

| 并发数 | 总耗时 | 吞吐量 (资源/秒) | 提升 |
|--------|--------|-----------------|------|
| 1 (默认) | 1000 秒 | 1 | 基准 |
| 3 | 333 秒 | 3 | **3x** |
| 5 | 200 秒 | 5 | **5x** |
| 10 | 100 秒 | 10 | **10x** |
| 20 | 50 秒 | 20 | **20x** |

### 3.2 资源消耗

| 并发数 | CPU 使用 | 内存使用 | API 调用/秒 |
|--------|---------|---------|------------|
| 1 | 10% | 100 MB | 10 |
| 5 | 30% | 150 MB | 50 |
| 10 | 50% | 200 MB | 100 |
| 20 | 80% | 300 MB | 200 |

---

## 4. 最佳实践

### 4.1 ✅ 根据资源数量设置并发数

```go
// ✅ 正确：根据集群规模调整
// 小集群（< 100 资源）
MaxConcurrentReconciles: 3

// 中等集群（100-1000 资源）
MaxConcurrentReconciles: 10

// 大集群（> 1000 资源）
MaxConcurrentReconciles: 20
```

### 4.2 ✅ 从环境变量读取

```go
// ✅ 正确：可配置
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    concurrent := env.GetIntEnvWithDefault("DEVBOX_CONCURRENT", 10)
    
    return ctrl.NewControllerManagedBy(mgr).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: concurrent,
        }).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}

// ❌ 错误：硬编码
MaxConcurrentReconciles: 10
```

### 4.3 ✅ 确保 Reconcile 逻辑线程安全

```go
// ✅ 正确：每个 Reconcile 使用独立的对象
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}  // 局部变量，线程安全
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    // ...
}

// ❌ 错误：共享状态，非线程安全
type DevboxReconciler struct {
    client.Client
    currentDevbox *devboxv1alpha2.Devbox  // 共享状态，并发不安全
}

func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    r.currentDevbox = &devboxv1alpha2.Devbox{}  // 竞态条件
    if err := r.Get(ctx, req.NamespacedName, r.currentDevbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    // ...
}
```

### 4.4 ✅ 监控并发性能

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
    reconcileDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "controller_reconcile_duration_seconds",
            Help: "Reconcile duration in seconds",
        },
        []string{"controller"},
    )
    activeReconciles = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "controller_active_reconciles",
            Help: "Number of active reconciles",
        },
        []string{"controller"},
    )
)

func init() {
    metrics.Registry.MustRegister(reconcileDuration, activeReconciles)
}

func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    start := time.Now()
    activeReconciles.WithLabelValues("devbox").Inc()
    defer func() {
        activeReconciles.WithLabelValues("devbox").Dec()
        reconcileDuration.WithLabelValues("devbox").Observe(time.Since(start).Seconds())
    }()
    
    // Reconcile 逻辑
    // ...
}
```

---

## 5. 实际案例

### 案例 1：Devbox 控制器

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    if err := mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &corev1.Pod{},
        devboxv1alpha2.PodNodeNameIndex,
        func(rawObj client.Object) []string {
            pod, _ := rawObj.(*corev1.Pod)
            if pod.Spec.NodeName == "" {
                return nil
            }
            return []string{pod.Spec.NodeName}
        },
    ); err != nil {
        return fmt.Errorf("failed to index field: %w", err)
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 10,  // 并发处理 10 个 Devbox
        }).
        For(&devboxv1alpha2.Devbox{}, builder.WithPredicates(predicate.Or(
            predicate.GenerationChangedPredicate{},
            NetworkTypeChangedPredicate{},
            ContentIDChangedPredicate{},
            LastContainerStatusChangedPredicate{},
            PhaseChangedPredicate{},
        ))).
        Owns(&corev1.Pod{}, builder.WithPredicates(
            predicate.ResourceVersionChangedPredicate{},
        )).
        Owns(&corev1.Service{}, builder.WithPredicates(
            predicate.GenerationChangedPredicate{},
        )).
        Owns(&corev1.Secret{}, builder.WithPredicates(
            predicate.GenerationChangedPredicate{},
        )).
        Complete(r)
}
```

**效果**：
- 可同时处理 10 个 Devbox 的创建/更新/删除
- 吞吐量提升 10 倍
- 配合 Predicate 减少不必要的 Reconcile

### 案例 2：User 控制器（动态并发）

```go
// controllers/user/controllers/user_controller.go
func (r *UserReconciler) SetupWithManager(mgr ctrl.Manager, opts ratelimiter.Options) error {
    r.Logger = ctrl.Log.WithName("user-controller")
    r.Recorder = mgr.GetEventRecorderFor("user-controller")
    r.finalizer = finalizer.NewFinalizer(r.Client, "sealos.io/user.finalizers")
    
    // 从配置读取并发数
    concurrent := ratelimiter.GetConcurrent(opts)
    r.Logger.Info("setup user controller", "concurrent", concurrent)
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&userv1.User{}, builder.WithPredicates(
            NewControllerRestartPredicate(10 * time.Minute),
        )).
        Owns(&v1.Namespace{}).
        Owns(&v1.ServiceAccount{}).
        WithOptions(kubecontroller.Options{
            MaxConcurrentReconciles: concurrent,
            RateLimiter:             ratelimiter.GetRateLimiter(opts),
        }).
        Complete(r)
}
```

### 案例 3：Account 控制器（环境变量配置）

```go
// controllers/account/main.go
func main() {
    // 从环境变量读取并发数
    concurrent := env.GetIntEnvWithDefault("ACCOUNT_CONCURRENT", 10)
    setupLog.Info("account controller config", "concurrent", concurrent)
    
    rateLimiterOptions := workqueue.RateLimiterOptions{
        BaseDelay:       time.Second,
        MaxDelay:        1000 * time.Second,
        FailureMaxDelay: 1000 * time.Second,
    }
    
    rateOpts := controller.Options{
        MaxConcurrentReconciles: concurrent,
        RateLimiter: workqueue.NewItemExponentialFailureRateLimiter(
            rateLimiterOptions.BaseDelay,
            rateLimiterOptions.MaxDelay,
        ),
    }
    
    if err = (&controllers.AccountReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr, rateOpts); err != nil {
        setupLog.Error(err, "unable to create controller")
        os.Exit(1)
    }
}
```

---

## 6. 注意事项

### 6.1 ⚠️ API Server 压力

```go
// ⚠️ 注意：高并发会增加 API Server 压力
MaxConcurrentReconciles: 50  // 可能导致 API Server 限流

// ✅ 建议：配合 RateLimiter 使用
WithOptions(controller.Options{
    MaxConcurrentReconciles: 20,
    RateLimiter: workqueue.NewItemExponentialFailureRateLimiter(
        time.Second,      // BaseDelay
        1000*time.Second, // MaxDelay
    ),
})
```

### 6.2 ⚠️ 资源竞争

```go
// ⚠️ 注意：多个 Worker 可能同时更新同一资源
// 场景：Devbox-1 的 Pod 变化触发 2 次 Reconcile

Worker 1: 更新 Devbox-1 Status
Worker 2: 同时更新 Devbox-1 Status
结果：409 Conflict

// ✅ 解决：使用 RetryOnConflict
err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &Devbox{}
    r.Get(ctx, key, latest)
    latest.Status.Phase = "Running"
    return r.Status().Update(ctx, latest)
})
```

### 6.3 ⚠️ 内存消耗

```go
// ⚠️ 注意：每个并发 Worker 都会占用内存
// 假设每个 Reconcile 占用 10 MB

MaxConcurrentReconciles: 1  → 10 MB
MaxConcurrentReconciles: 10 → 100 MB
MaxConcurrentReconciles: 50 → 500 MB

// ✅ 建议：监控内存使用，避免 OOM
```

---

## 7. 并发数选择指南

### 7.1 根据资源数量

| 资源数量 | 推荐并发数 | 说明 |
|---------|-----------|------|
| < 50 | 1-3 | 串行足够 |
| 50-200 | 3-5 | 小规模并发 |
| 200-1000 | 5-10 | 中等并发 |
| 1000-5000 | 10-20 | 高并发 |
| > 5000 | 20-50 | 超高并发（需要监控） |

### 7.2 根据 Reconcile 耗时

| 平均耗时 | 推荐并发数 | 说明 |
|---------|-----------|------|
| < 100ms | 3-5 | 快速操作 |
| 100-500ms | 5-10 | 中等耗时 |
| 500ms-2s | 10-20 | 较慢操作 |
| > 2s | 20-50 | 慢操作（需要优化） |

### 7.3 根据资源类型

| 资源类型 | 推荐并发数 | 说明 |
|---------|-----------|------|
| 轻量级（ConfigMap） | 5-10 | 快速处理 |
| 中等（Deployment） | 10-20 | 需要创建多个子资源 |
| 重量级（复杂 CRD） | 3-10 | 复杂逻辑，避免过载 |

---

## 8. 总结

### 核心要点

1. ✅ **根据场景设置并发数** - 不是越大越好
2. ✅ **从环境变量读取** - 便于调整
3. ✅ **确保线程安全** - 避免共享状态
4. ✅ **配合 RateLimiter** - 保护 API Server
5. ✅ **监控性能** - 观察吞吐量和资源消耗

### 适用场景

| 场景 | 是否使用并发 | 推荐并发数 |
|------|------------|-----------|
| 大量资源（> 100） | ✅ 是 | 10-20 |
| 少量资源（< 50） | ❌ 否 | 1-3 |
| 快速操作（< 100ms） | ✅ 是 | 5-10 |
| 慢操作（> 2s） | ✅ 是 | 10-20 |
| API 密集型 | ⚠️ 谨慎 | 5-10 + RateLimiter |

### 与其他模式的配合

- **Custom Predicate**: 减少 Reconcile 次数，降低并发压力
- **Field Indexer**: 加速查询，提升并发效率
- **RetryOnConflict**: 处理并发冲突

### 参考资源

- [Controller Options 文档](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/controller#Options)
- [Workqueue RateLimiter](https://pkg.go.dev/k8s.io/client-go/util/workqueue)
- [Sealos Devbox Controller 实现](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
