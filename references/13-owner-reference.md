# Owner Reference 模式

**复杂度**: 🟢 简单  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 级联删除、资源关系管理、垃圾回收

## 概述

Owner Reference 模式通过在子资源的 `metadata.ownerReferences` 中设置父资源的引用，实现 Kubernetes 的自动级联删除和垃圾回收机制。

## 快速开始

```go
// 创建 Service 并设置 Owner Reference
service := &corev1.Service{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-service",
        Namespace: "default",
    },
}

_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    service.Spec.Type = corev1.ServiceTypeClusterIP
    service.Spec.Ports = ports
    
    // 设置 Owner Reference
    return controllerutil.SetControllerReference(terminal, service, r.Scheme)
})
```

**效果**：删除 Terminal → 自动删除 Service

---

## 1. 工作原理

### 1.1 级联删除流程

```
用户执行: kubectl delete terminal my-terminal
         │
         ▼
┌────────────────────────────────────────────┐
│ API Server 删除 Terminal                   │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│ Garbage Collector 检查 ownerReferences     │
│ 查找所有 Owner 为 my-terminal 的资源      │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│ 自动删除子资源                              │
│ - Deployment                               │
│ - Service                                  │
│ - Ingress                                  │
└────────────────────────────────────────────┘
```

### 1.2 OwnerReference 结构

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
  ownerReferences:
  - apiVersion: terminal.sealos.io/v1
    kind: Terminal
    name: my-terminal
    uid: 12345-67890-abcde
    controller: true              # 表示这是 Controller Owner
    blockOwnerDeletion: true      # 阻止删除 Owner（直到子资源删除）
spec:
  # ...
```

---

## 2. 使用方法

### 2.1 基本用法

```go
// controllers/terminal/controllers/terminal_controller.go
func (r *TerminalReconciler) syncService(
    ctx context.Context,
    terminal *terminalv1.Terminal,
    recLabels map[string]string,
) error {
    service := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      terminal.Status.ServiceName,
            Namespace: terminal.Namespace,
        },
    }
    
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
        service.Spec.Selector = recLabels
        service.Spec.Type = corev1.ServiceTypeClusterIP
        service.Spec.Ports = []corev1.ServicePort{{
            Name: "http", Port: 8080, TargetPort: intstr.FromInt(8080),
        }}
        
        // 设置 Owner Reference
        return controllerutil.SetControllerReference(terminal, service, r.Scheme)
    })
    
    return err
}
```

### 2.2 多个子资源

```go
func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &terminalv1.Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    recLabels := label.RecommendedLabels(&label.Recommended{
        Name:      terminal.Name,
        ManagedBy: "terminal-controller",
        PartOf:    "terminal",
    })
    
    // 所有子资源都设置 Owner Reference
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
    
    // 删除 Terminal 时，Deployment、Service、Ingress 自动删除
    return ctrl.Result{}, nil
}
```

---

## 3. 限制和解决方案

### 3.1 同 Namespace 限制

```go
// ❌ 错误：跨 Namespace 设置 Owner Reference
owner := &Terminal{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-terminal",
        Namespace: "ns-a",
    },
}

service := &corev1.Service{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-service",
        Namespace: "ns-b",  // ❌ 不同 Namespace
    },
}

controllerutil.SetControllerReference(owner, service, r.Scheme)
// 错误: cross-namespace owner references are disallowed

// ✅ 解决方案：使用 Label 关联
service := &corev1.Service{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-service",
        Namespace: "ns-b",
        Labels: map[string]string{
            "terminal.sealos.io/name":      owner.Name,
            "terminal.sealos.io/namespace": owner.Namespace,
        },
    },
}

// 使用 Finalizer 手动清理
func (r *TerminalReconciler) cleanup(ctx context.Context, terminal *Terminal) error {
    // 查找并删除跨 Namespace 的资源
    serviceList := &corev1.ServiceList{}
    if err := r.List(ctx, serviceList,
        client.MatchingLabels{
            "terminal.sealos.io/name":      terminal.Name,
            "terminal.sealos.io/namespace": terminal.Namespace,
        },
    ); err != nil {
        return err
    }
    
    for _, svc := range serviceList.Items {
        if err := r.Delete(ctx, &svc); err != nil {
            return client.IgnoreNotFound(err)
        }
    }
    
    return nil
}
```

### 3.2 Cluster-scoped 资源限制

```go
// ❌ 错误：Namespace-scoped 资源不能 Own Cluster-scoped 资源
owner := &Devbox{  // Namespace-scoped
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-devbox",
        Namespace: "user-ns",
    },
}

node := &corev1.Node{  // Cluster-scoped
    ObjectMeta: metav1.ObjectMeta{
        Name: "node-1",
    },
}

controllerutil.SetControllerReference(owner, node, r.Scheme)
// 错误: namespace-scoped owner cannot own cluster-scoped resource

// ✅ 解决方案：使用 Label + Finalizer
node := &corev1.Node{
    ObjectMeta: metav1.ObjectMeta{
        Name: "node-1",
        Labels: map[string]string{
            "devbox.sealos.io/owner": "user-ns/my-devbox",
        },
    },
}
```

---

## 4. 实际案例

### 案例 1：Terminal 控制器的完整资源树

```go
// Terminal (Owner)
//   ├─ Deployment (Owned)
//   ├─ Service (Owned)
//   └─ Ingress (Owned)

// controllers/terminal/controllers/terminal_controller.go
func (r *TerminalReconciler) syncDeployment(
    ctx context.Context,
    terminal *terminalv1.Terminal,
    hostname *string,
    recLabels map[string]string,
) error {
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      terminal.Name,
            Namespace: terminal.Namespace,
        },
    }
    
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, deployment, func() error {
        deployment.Labels = recLabels
        deployment.Spec.Replicas = ptr.To(int32(1))
        deployment.Spec.Selector = &metav1.LabelSelector{
            MatchLabels: recLabels,
        }
        deployment.Spec.Template.Labels = recLabels
        deployment.Spec.Template.Spec.Containers = []corev1.Container{{
            Name:  "terminal",
            Image: terminal.Spec.Image,
        }}
        
        if *hostname == "" {
            *hostname = deployment.Spec.Template.Spec.Hostname
        }
        
        return controllerutil.SetControllerReference(terminal, deployment, r.Scheme)
    })
    
    return err
}
```

---

## 5. 总结

### 核心要点

1. ✅ **自动级联删除** - 无需手动清理子资源
2. ✅ **使用 SetControllerReference** - 自动设置正确的字段
3. ✅ **同 Namespace 限制** - Owner 和 Owned 必须在同一 Namespace
4. ✅ **配合 CreateOrUpdate** - 在 Mutate 函数中设置
5. ✅ **跨 Namespace 使用 Label** - 配合 Finalizer 手动清理

### 适用场景

| 场景 | 是否使用 Owner Reference |
|------|----------------------|
| 同 Namespace 子资源 | ✅ 是 |
| 跨 Namespace 资源 | ❌ 否（使用 Label + Finalizer） |
| Cluster-scoped 资源 | ⚠️ 有限制 |

### 与其他模式的配合

- **CreateOrUpdate**: 在 Mutate 函数中设置 Owner Reference
- **Finalizer**: 清理无法使用 Owner Reference 的资源

### 参考资源

- [Owner References 文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
- [Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/)
- [Sealos Terminal Controller](https://github.com/labring/sealos/blob/main/controllers/terminal/controllers/terminal_controller.go)
