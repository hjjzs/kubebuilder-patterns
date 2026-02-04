# Recommended Labels 模式

**复杂度**: 🟢 简单  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 资源标识、查询过滤、监控指标

## 概述

Recommended Labels 模式遵循 Kubernetes 推荐的标签规范，为资源添加标准化的标签，便于识别、查询、监控和管理。

## 快速开始

```go
import "github.com/labring/sealos/controllers/pkg/utils/label"

// 生成推荐标签
recLabels := label.RecommendedLabels(&label.Recommended{
    Name:      "my-terminal",
    ManagedBy: "terminal-controller",
    PartOf:    "terminal",
})

// 结果:
// {
//   "app.kubernetes.io/name": "my-terminal",
//   "app.kubernetes.io/managed-by": "terminal-controller",
//   "app.kubernetes.io/part-of": "terminal",
// }

// 应用到资源
service := &corev1.Service{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-service",
        Namespace: "default",
        Labels:    recLabels,
    },
}
```

---

## 1. 标准标签

### 1.1 Kubernetes 推荐标签

| 标签 | 说明 | 示例 |
|------|------|------|
| `app.kubernetes.io/name` | 应用名称 | `terminal`, `devbox` |
| `app.kubernetes.io/instance` | 应用实例 | `my-terminal-001` |
| `app.kubernetes.io/version` | 应用版本 | `v1.0.0` |
| `app.kubernetes.io/component` | 组件类型 | `database`, `frontend` |
| `app.kubernetes.io/part-of` | 所属应用 | `terminal-system` |
| `app.kubernetes.io/managed-by` | 管理者 | `terminal-controller` |

### 1.2 Sealos 实现

```go
// controllers/pkg/utils/label/recommend.go
type Recommended struct {
    Name      string
    Instance  string
    Version   string
    Component string
    PartOf    string
    ManagedBy string
}

func (r *Recommended) Labels() map[string]string {
    ret := map[string]string{}
    
    if r.Name != "" {
        ret[AppName] = r.Name
    }
    if r.Instance != "" {
        ret[AppInstance] = r.Instance
    }
    if r.Version != "" {
        ret[AppVersion] = r.Version
    }
    if r.Component != "" {
        ret[AppComponent] = r.Component
    }
    if r.PartOf != "" {
        ret[AppPartOf] = r.PartOf
    }
    if r.ManagedBy != "" {
        ret[AppManagedBy] = r.ManagedBy
    }
    
    return ret
}

func RecommendedLabels(r *Recommended) map[string]string {
    return r.Labels()
}
```

---

## 2. 使用方法

### 2.1 基本使用

```go
// controllers/terminal/controllers/terminal_controller.go
func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &terminalv1.Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 生成推荐标签
    recLabels := label.RecommendedLabels(&label.Recommended{
        Name:      terminal.Name,
        ManagedBy: label.DefaultManagedBy,  // "sealos-controller"
        PartOf:    "terminal",
    })
    
    // 应用到所有子资源
    if err := r.syncDeployment(ctx, terminal, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    
    if err := r.syncService(ctx, terminal, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

### 2.2 添加自定义标签

```go
// ✅ 正确：在推荐标签基础上添加自定义标签
recLabels := label.RecommendedLabels(&label.Recommended{
    Name:      terminal.Name,
    ManagedBy: "terminal-controller",
    PartOf:    "terminal",
})

// 添加自定义标签
recLabels["terminal.sealos.io/user"] = terminal.Namespace
recLabels["terminal.sealos.io/type"] = terminal.Spec.Type

// 应用到资源
service := &corev1.Service{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-service",
        Namespace: "default",
        Labels:    recLabels,
    },
}
```

### 2.3 用于资源查询

```go
// 查询特定应用的所有资源
podList := &corev1.PodList{}
if err := r.List(ctx, podList,
    client.InNamespace("user-ns"),
    client.MatchingLabels{
        "app.kubernetes.io/name": "my-terminal",
    },
); err != nil {
    return err
}

// 查询特定控制器管理的所有资源
serviceList := &corev1.ServiceList{}
if err := r.List(ctx, serviceList,
    client.MatchingLabels{
        "app.kubernetes.io/managed-by": "terminal-controller",
    },
); err != nil {
    return err
}
```

---

## 3. 实际案例

### 案例 1：Terminal 控制器

```go
// controllers/terminal/controllers/terminal_controller.go
const TerminalPartOf = "terminal"

func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &terminalv1.Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    recLabels := label.RecommendedLabels(&label.Recommended{
        Name:      terminal.Name,
        ManagedBy: label.DefaultManagedBy,
        PartOf:    TerminalPartOf,
    })
    
    // 向后兼容：旧版本使用的标签
    recLabels["TerminalID"] = terminal.Name
    
    // 应用到所有子资源
    // ...
}
```

### 案例 2：Devbox 控制器

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    recLabels := map[string]string{
        "app.kubernetes.io/name":       devbox.Name,
        "app.kubernetes.io/managed-by": "devbox-controller",
        "app.kubernetes.io/part-of":    "devbox",
        "app.kubernetes.io/instance":   devbox.Name,
    }
    
    // 添加 Devbox 特定标签
    recLabels["devbox.sealos.io/name"] = devbox.Name
    
    // 用于查询和过滤
    // ...
}
```

---

## 4. 最佳实践

### 4.1 ✅ 使用推荐标签

```go
// ✅ 正确：使用 Kubernetes 推荐标签
recLabels := label.RecommendedLabels(&label.Recommended{
    Name:      "my-app",
    ManagedBy: "my-controller",
    PartOf:    "my-system",
})

// ❌ 错误：使用自定义标签
labels := map[string]string{
    "app": "my-app",           // 不标准
    "controller": "my-controller",
}
```

### 4.2 ✅ 标签用于选择器

```go
// ✅ 正确：使用标签作为 Selector
service := &corev1.Service{
    Spec: corev1.ServiceSpec{
        Selector: recLabels,  // 使用相同的标签
    },
}

deployment := &appsv1.Deployment{
    Spec: appsv1.DeploymentSpec{
        Selector: &metav1.LabelSelector{
            MatchLabels: recLabels,
        },
        Template: corev1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{
                Labels: recLabels,  // Pod 也使用相同标签
            },
        },
    },
}
```

### 4.3 ✅ 标签用于监控

```go
// Prometheus 监控规则
- record: terminal_count
  expr: count(kube_pod_labels{app_kubernetes_io_managed_by="terminal-controller"})

- record: devbox_count_by_phase
  expr: count by (phase) (kube_pod_labels{app_kubernetes_io_managed_by="devbox-controller"})
```

---

## 5. 总结

### 核心要点

1. ✅ **使用推荐标签** - 遵循 Kubernetes 规范
2. ✅ **标签用于选择器** - Service、Deployment 等
3. ✅ **标签用于查询** - List 操作
4. ✅ **标签用于监控** - Prometheus 指标
5. ✅ **向后兼容** - 保留旧标签

### 常用标签

| 场景 | 必需标签 | 可选标签 |
|------|---------|---------|
| 基本应用 | name, managed-by | part-of |
| 多实例应用 | name, instance | version |
| 微服务 | name, component, part-of | version |

### 与其他模式的配合

- **CreateOrUpdate**: 创建资源时设置标签
- **Resource Deletion**: 按标签批量删除
- **Custom Predicate**: 按标签过滤事件

### 参考资源

- [Kubernetes Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [Sealos Label Package](https://github.com/labring/sealos/tree/main/controllers/pkg/utils/label)
- [Sealos Terminal Controller](https://github.com/labring/sealos/blob/main/controllers/terminal/controllers/terminal_controller.go)
