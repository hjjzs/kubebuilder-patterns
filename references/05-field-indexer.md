# Field Indexer 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 优化 List 查询、缓存索引、跨资源关联

## 概述

Field Indexer 是 controller-runtime 提供的缓存索引机制。通过为资源的特定字段建立索引，可以将 O(n) 的全量扫描优化为 O(1) 的索引查找，显著提升 List 操作的性能。

## 快速开始

### 问题场景

```go
// ❌ 低效：全量扫描所有 Pod，过滤出特定 Node 上的 Pod
podList := &corev1.PodList{}
if err := r.List(ctx, podList); err != nil {
    return err
}

var podsOnNode []*corev1.Pod
for _, pod := range podList.Items {
    if pod.Spec.NodeName == targetNode {
        podsOnNode = append(podsOnNode, &pod)
    }
}
```

**性能问题**：
- 需要扫描所有 Pod（可能数千个）
- 每次查询都要遍历整个列表
- 时间复杂度：O(n)

### 使用 Field Indexer

```go
// 1. 在 SetupWithManager 中注册索引
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    if err := mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &corev1.Pod{},
        "spec.nodeName",  // 索引字段
        func(rawObj client.Object) []string {
            pod := rawObj.(*corev1.Pod)
            if pod.Spec.NodeName == "" {
                return nil
            }
            return []string{pod.Spec.NodeName}
        },
    ); err != nil {
        return err
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}

// 2. 使用索引查询
podList := &corev1.PodList{}
if err := r.List(ctx, podList,
    client.MatchingFields{"spec.nodeName": targetNode},
); err != nil {
    return err
}
// 直接返回目标 Node 上的 Pod，时间复杂度：O(1)
```

---

## 1. 工作原理

### 1.1 索引结构

```
┌─────────────────────────────────────────────────────────┐
│                    Field Indexer                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Index: "spec.nodeName"                                 │
│                                                         │
│  ┌──────────────┬────────────────────────────────┐     │
│  │ Node Name    │ Pods                           │     │
│  ├──────────────┼────────────────────────────────┤     │
│  │ node-1       │ [pod-a, pod-b, pod-c]          │     │
│  │ node-2       │ [pod-d, pod-e]                 │     │
│  │ node-3       │ [pod-f, pod-g, pod-h, pod-i]   │     │
│  └──────────────┴────────────────────────────────┘     │
│                                                         │
│  Index: "metadata.namespace"                            │
│                                                         │
│  ┌──────────────┬────────────────────────────────┐     │
│  │ Namespace    │ Pods                           │     │
│  ├──────────────┼────────────────────────────────┤     │
│  │ ns-user1     │ [pod-a, pod-b]                 │     │
│  │ ns-user2     │ [pod-c, pod-d, pod-e]          │     │
│  └──────────────┴────────────────────────────────┘     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 1.2 查询流程

```
r.List(ctx, podList, client.MatchingFields{"spec.nodeName": "node-1"})
                │
                ▼
┌───────────────────────────────────────────────────┐
│ 1. 检查是否有 "spec.nodeName" 索引                │
└───────────────┬───────────────────────────────────┘
                │ 有索引
                ▼
┌───────────────────────────────────────────────────┐
│ 2. 从索引中查找 "node-1" 对应的 Pod 列表          │
│    返回: [pod-a, pod-b, pod-c]                    │
└───────────────┬───────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────┐
│ 3. 返回结果（O(1) 时间复杂度）                    │
└───────────────────────────────────────────────────┘
```

---

## 2. 使用方法

### 2.1 注册索引

```go
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // 为 Pod 的 NodeName 字段建立索引
    if err := mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &corev1.Pod{},                    // 要索引的资源类型
        devboxv1alpha2.PodNodeNameIndex,  // 索引名称（自定义）
        func(rawObj client.Object) []string {  // 索引函数
            pod := rawObj.(*corev1.Pod)
            if pod.Spec.NodeName == "" {
                return nil  // 返回 nil 表示不索引此对象
            }
            return []string{pod.Spec.NodeName}  // 返回索引值
        },
    ); err != nil {
        return fmt.Errorf("failed to index field: %w", err)
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}
```

**索引名称定义**：

```go
// controllers/devbox/api/v1alpha2/devbox_types.go
const (
    PodNodeNameIndex = "spec.nodeName"
)
```

### 2.2 使用索引查询

```go
func (r *DevboxReconciler) getPodsOnNode(ctx context.Context, nodeName string) ([]*corev1.Pod, error) {
    podList := &corev1.PodList{}
    
    // 使用索引查询
    if err := r.List(ctx, podList,
        client.MatchingFields{devboxv1alpha2.PodNodeNameIndex: nodeName},
    ); err != nil {
        return nil, err
    }
    
    pods := make([]*corev1.Pod, len(podList.Items))
    for i := range podList.Items {
        pods[i] = &podList.Items[i]
    }
    return pods, nil
}
```

### 2.3 多条件查询

```go
// 结合 Namespace 和索引查询
podList := &corev1.PodList{}
if err := r.List(ctx, podList,
    client.InNamespace("user-ns"),  // Namespace 过滤
    client.MatchingFields{devboxv1alpha2.PodNodeNameIndex: nodeName},  // 索引查询
    client.MatchingLabels{"app": "devbox"},  // Label 过滤
); err != nil {
    return err
}
```

---

## 3. 实际案例

### 3.1 案例 1：Devbox 控制器的 Node 资源统计

Devbox 需要统计每个 Node 上的资源使用情况：

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // 注册 Pod NodeName 索引
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
        return fmt.Errorf("failed to index field %s: %w", devboxv1alpha2.PodNodeNameIndex, err)
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}

// 计算 Node 上的 CPU 使用率
func (r *DevboxReconciler) getTotalCPURequestRatio(ctx context.Context) (float64, error) {
    podList := &corev1.PodList{}
    
    // 使用索引查询当前 Node 上的所有 Pod
    if err := r.List(ctx, podList,
        client.MatchingFields{devboxv1alpha2.PodNodeNameIndex: r.NodeName},
    ); err != nil {
        return 0, err
    }
    
    var totalCPURequest int64
    for _, pod := range podList.Items {
        for _, container := range pod.Spec.Containers {
            if cpuReq, ok := container.Resources.Requests[corev1.ResourceCPU]; ok {
                totalCPURequest += cpuReq.MilliValue()
            }
        }
    }
    
    // 获取 Node 的可分配 CPU
    node := &corev1.Node{}
    if err := r.Get(ctx, client.ObjectKey{Name: r.NodeName}, node); err != nil {
        return 0, err
    }
    
    allocatableCPU := node.Status.Allocatable[corev1.ResourceCPU]
    ratio := float64(totalCPURequest) / float64(allocatableCPU.MilliValue())
    
    return ratio, nil
}
```

**性能提升**：
- 无索引：扫描所有 Pod（可能 1000+），过滤出目标 Node 的 Pod
- 有索引：直接返回目标 Node 的 Pod（可能 10-50 个）
- **性能提升：100x**

### 3.2 案例 2：Account 控制器的 Namespace Owner 索引

Account 控制器需要查找用户拥有的所有 Namespace：

```go
// controllers/account/controllers/cache/cache.go
func SetupCache(mgr ctrl.Manager) error {
    ns := &corev1.Namespace{}
    
    // Namespace Name 索引
    nsNameFunc := func(obj client.Object) []string {
        _ns, ok := obj.(*corev1.Namespace)
        if !ok {
            return nil
        }
        return []string{_ns.Name}
    }
    
    // Namespace Owner 索引
    nsOwnerFunc := func(obj client.Object) []string {
        _ns, ok := obj.(*corev1.Namespace)
        if !ok {
            return nil
        }
        owner := _ns.Labels[v1.UserLabelOwnerKey]
        if owner == "" {
            return nil
        }
        return []string{owner}
    }
    
    // 注册索引
    for _, idx := range []struct {
        obj          client.Object
        field        string
        extractValue client.IndexerFunc
    }{
        {ns, accountv1.Name, nsNameFunc},
        {ns, accountv1.Owner, nsOwnerFunc},
    } {
        if err := mgr.GetFieldIndexer().IndexField(
            context.TODO(),
            idx.obj,
            idx.field,
            idx.extractValue,
        ); err != nil {
            return err
        }
    }
    
    return nil
}
```

**使用索引查询用户的 Namespace**：

```go
func (r *AccountReconciler) getUserNamespaces(ctx context.Context, username string) ([]*corev1.Namespace, error) {
    nsList := &corev1.NamespaceList{}
    
    // 使用 Owner 索引查询
    if err := r.List(ctx, nsList,
        client.MatchingFields{accountv1.Owner: username},
    ); err != nil {
        return nil, err
    }
    
    namespaces := make([]*corev1.Namespace, len(nsList.Items))
    for i := range nsList.Items {
        namespaces[i] = &nsList.Items[i]
    }
    return namespaces, nil
}
```

### 3.3 案例 3：Resources 控制器的多索引查询

```go
// controllers/resources/main.go
func setupIndexers(mgr ctrl.Manager) error {
    // Pod OwnerReferences 索引
    if err := mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &corev1.Pod{},
        "ownerReferences.uid",
        func(obj client.Object) []string {
            pod := obj.(*corev1.Pod)
            var uids []string
            for _, ref := range pod.OwnerReferences {
                uids = append(uids, string(ref.UID))
            }
            return uids
        },
    ); err != nil {
        return err
    }
    
    // Pod Namespace 索引（虽然内置，但可以自定义）
    if err := mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &corev1.Pod{},
        "metadata.namespace",
        func(obj client.Object) []string {
            return []string{obj.GetNamespace()}
        },
    ); err != nil {
        return err
    }
    
    return nil
}
```

---

## 4. 高级用法

### 4.1 多值索引

一个对象可以返回多个索引值：

```go
// 为 Pod 的所有 OwnerReferences 建立索引
mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &corev1.Pod{},
    "ownerReferences.uid",
    func(rawObj client.Object) []string {
        pod := rawObj.(*corev1.Pod)
        var uids []string
        for _, ref := range pod.OwnerReferences {
            uids = append(uids, string(ref.UID))
        }
        return uids  // 返回多个值
    },
)

// 查询：查找某个 Owner 的所有 Pod
podList := &corev1.PodList{}
r.List(ctx, podList,
    client.MatchingFields{"ownerReferences.uid": string(ownerUID)},
)
```

### 4.2 复合索引

虽然 Field Indexer 不直接支持复合索引，但可以通过字符串拼接实现：

```go
// 为 Namespace + Name 建立复合索引
mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &corev1.Pod{},
    "namespace-name",
    func(rawObj client.Object) []string {
        pod := rawObj.(*corev1.Pod)
        // 使用 "/" 分隔 Namespace 和 Name
        return []string{pod.Namespace + "/" + pod.Name}
    },
)

// 查询
podList := &corev1.PodList{}
r.List(ctx, podList,
    client.MatchingFields{"namespace-name": "user-ns/my-pod"},
)
```

### 4.3 条件索引

只为满足条件的对象建立索引：

```go
// 只为 Running 状态的 Pod 建立索引
mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &corev1.Pod{},
    "spec.nodeName",
    func(rawObj client.Object) []string {
        pod := rawObj.(*corev1.Pod)
        
        // 只索引 Running 状态的 Pod
        if pod.Status.Phase != corev1.PodRunning {
            return nil
        }
        
        if pod.Spec.NodeName == "" {
            return nil
        }
        return []string{pod.Spec.NodeName}
    },
)
```

---

## 5. 性能对比

### 5.1 查询性能

**场景**：集群中有 10000 个 Pod，分布在 100 个 Node 上

| 操作 | 无索引 | 有索引 | 提升 |
|------|--------|--------|------|
| 查询单个 Node 的 Pod | 10ms | 0.1ms | **100x** |
| 查询 10 个 Node 的 Pod | 100ms | 1ms | **100x** |
| 统计所有 Node 的资源 | 1000ms | 10ms | **100x** |

### 5.2 内存开销

**索引内存开销**：

```
每个索引项 ≈ 100 bytes（索引值 + 对象指针）
10000 个 Pod × 100 bytes = 1MB

总内存开销 ≈ 索引数量 × 1MB
```

**结论**：内存开销很小，性能提升巨大。

---

## 6. 最佳实践

### 6.1 ✅ 为高频查询的字段建立索引

```go
// ✅ 正确：为经常查询的字段建立索引
// 场景：频繁查询某个 Node 上的 Pod
mgr.GetFieldIndexer().IndexField(ctx, &corev1.Pod{}, "spec.nodeName", ...)

// ❌ 错误：为低频查询的字段建立索引（浪费内存）
// 场景：只查询一次
mgr.GetFieldIndexer().IndexField(ctx, &corev1.Pod{}, "spec.hostname", ...)
```

### 6.2 ✅ 索引名称使用常量

```go
// ✅ 正确：使用常量
const PodNodeNameIndex = "spec.nodeName"

mgr.GetFieldIndexer().IndexField(ctx, &corev1.Pod{}, PodNodeNameIndex, ...)
r.List(ctx, podList, client.MatchingFields{PodNodeNameIndex: nodeName})

// ❌ 错误：使用字符串字面量（容易拼写错误）
mgr.GetFieldIndexer().IndexField(ctx, &corev1.Pod{}, "spec.nodeName", ...)
r.List(ctx, podList, client.MatchingFields{"spec.nodeNmae": nodeName})  // 拼写错误
```

### 6.3 ✅ 处理空值

```go
// ✅ 正确：返回 nil 表示不索引
mgr.GetFieldIndexer().IndexField(ctx, &corev1.Pod{}, "spec.nodeName",
    func(rawObj client.Object) []string {
        pod := rawObj.(*corev1.Pod)
        if pod.Spec.NodeName == "" {
            return nil  // 不索引未调度的 Pod
        }
        return []string{pod.Spec.NodeName}
    },
)

// ❌ 错误：返回空字符串（会索引为 ""）
mgr.GetFieldIndexer().IndexField(ctx, &corev1.Pod{}, "spec.nodeName",
    func(rawObj client.Object) []string {
        pod := rawObj.(*corev1.Pod)
        return []string{pod.Spec.NodeName}  // 可能返回 [""]
    },
)
```

### 6.4 ✅ 在 SetupWithManager 中注册索引

```go
// ✅ 正确：在 SetupWithManager 中注册
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    if err := mgr.GetFieldIndexer().IndexField(...); err != nil {
        return err
    }
    return ctrl.NewControllerManagedBy(mgr).For(...).Complete(r)
}

// ❌ 错误：在 Reconcile 中注册（每次 Reconcile 都注册）
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    mgr.GetFieldIndexer().IndexField(...)  // ❌ 错误位置
    // ...
}
```

---

## 7. 总结

### 核心要点

1. ✅ **为高频查询字段建立索引** - 显著提升性能
2. ✅ **使用常量定义索引名称** - 避免拼写错误
3. ✅ **处理空值** - 返回 nil 表示不索引
4. ✅ **在 SetupWithManager 中注册** - 避免重复注册
5. ✅ **结合其他过滤条件** - Namespace、Label 等

### 适用场景

| 场景 | 是否使用 Field Indexer |
|------|---------------------|
| 频繁按 Node 查询 Pod | ✅ 是 |
| 频繁按 Owner 查询资源 | ✅ 是 |
| 频繁按自定义字段查询 | ✅ 是 |
| 只查询一次 | ❌ 否 |
| 全量扫描 | ❌ 否 |

### 参考资源

- [Field Indexer 文档](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache)
- [Sealos Field Indexer 参考文档](../../.claude/skills/sealos-kubebuilder/references/field-indexer.md)
- [Sealos Devbox Controller 实现](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
