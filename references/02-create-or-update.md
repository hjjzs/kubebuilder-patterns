# CreateOrUpdate 模式

**复杂度**: 🟢 简单  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 子资源管理、声明式资源同步

## 概述

CreateOrUpdate 是 controller-runtime 提供的声明式资源管理模式。它实现了"如果不存在则创建，如果存在则更新"的幂等操作，是 Kubernetes 控制器管理子资源的标准方法。

## 目录

1. [问题背景](#1-问题背景)
2. [工作原理](#2-工作原理)
3. [使用方法](#3-使用方法)
4. [与 Owner Reference 配合](#4-与-owner-reference-配合)
5. [最佳实践](#5-最佳实践)
6. [常见错误](#6-常见错误)
7. [实际案例](#7-实际案例)
8. [性能优化](#8-性能优化)

---

## 1. 问题背景

### 1.1 传统的资源管理方式

**方式一：先检查再创建/更新**

```go
// ❌ 存在竞态条件
service := &corev1.Service{}
err := r.Get(ctx, key, service)
if apierrors.IsNotFound(err) {
    // 创建
    return r.Create(ctx, service)
} else if err == nil {
    // 更新
    return r.Update(ctx, service)
}
return err
```

**问题**：
- ❌ Get 和 Create 之间有时间窗口，可能被其他客户端创建
- ❌ 需要手动处理各种错误情况
- ❌ 代码冗长，容易出错

**方式二：直接创建，失败则更新**

```go
// ❌ 逻辑复杂
err := r.Create(ctx, service)
if apierrors.IsAlreadyExists(err) {
    existing := &corev1.Service{}
    if err := r.Get(ctx, key, existing); err != nil {
        return err
    }
    service.ResourceVersion = existing.ResourceVersion
    return r.Update(ctx, service)
}
return err
```

**问题**：
- ❌ 需要手动复制 ResourceVersion
- ❌ 可能覆盖其他控制器的修改
- ❌ 错误处理复杂

### 1.2 CreateOrUpdate 的优势

```go
// ✅ 简洁、安全、幂等
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    // 在这里设置期望状态
    service.Spec.Type = corev1.ServiceTypeClusterIP
    service.Spec.Ports = []corev1.ServicePort{...}
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})
```

**优势**：
- ✅ 自动处理创建和更新逻辑
- ✅ 原子操作，无竞态条件
- ✅ 只修改 Mutate 函数中指定的字段
- ✅ 代码简洁，易于维护

---

## 2. 工作原理

### 2.1 函数签名

```go
// sigs.k8s.io/controller-runtime/pkg/controller/controllerutil
func CreateOrUpdate(
    ctx context.Context,
    c client.Client,
    obj client.Object,
    f MutateFn,
) (OperationResult, error)

type MutateFn func() error

type OperationResult string

const (
    OperationResultNone             OperationResult = "unchanged"
    OperationResultCreated          OperationResult = "created"
    OperationResultUpdated          OperationResult = "updated"
    OperationResultUpdatedStatus    OperationResult = "updatedStatus"
    OperationResultUpdatedStatusOnly OperationResult = "updatedStatusOnly"
)
```

### 2.2 执行流程

```
┌────────────────────────────────────────────────────────────────┐
│ CreateOrUpdate(ctx, client, obj, mutateFn)                     │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ 1. 获取对象的 Key     │
              │    (Namespace + Name) │
              └──────────┬────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ 2. 尝试 Get 对象     │
              └──────────┬────────────┘
                         │
           ┌─────────────┴─────────────┐
           │                           │
           ▼ NotFound                  ▼ Found
    ┌──────────────┐          ┌──────────────────┐
    │ 3a. 执行     │          │ 3b. 保存原始对象 │
    │    mutateFn  │          │     的副本        │
    └──────┬───────┘          └────────┬─────────┘
           │                           │
           ▼                           ▼
    ┌──────────────┐          ┌──────────────────┐
    │ 4a. Create   │          │ 4b. 执行 mutateFn│
    │     对象     │          │     (修改对象)    │
    └──────┬───────┘          └────────┬─────────┘
           │                           │
           │                           ▼
           │                  ┌──────────────────┐
           │                  │ 5b. 对比修改前后 │
           │                  │     是否有变化    │
           │                  └────────┬─────────┘
           │                           │
           │              ┌────────────┴────────────┐
           │              │ 有变化                  │ 无变化
           │              ▼                         ▼
           │     ┌──────────────────┐      ┌──────────────┐
           │     │ 6b. Update 对象  │      │ 返回 None    │
           │     └────────┬─────────┘      └──────────────┘
           │              │
           ▼              ▼
    ┌──────────────────────────────┐
    │ 返回 OperationResult + error │
    └──────────────────────────────┘
```

### 2.3 Mutate 函数的作用

Mutate 函数定义了资源的**期望状态**：

```go
mutateFn := func() error {
    // 设置期望的字段值
    service.Spec.Type = corev1.ServiceTypeClusterIP
    service.Spec.Selector = map[string]string{"app": "myapp"}
    service.Spec.Ports = []corev1.ServicePort{{Port: 8080}}
    
    // 设置 Owner Reference
    return controllerutil.SetControllerReference(owner, service, scheme)
}
```

**关键特性**：
- ✅ 只修改你关心的字段
- ✅ 不会覆盖其他控制器管理的字段
- ✅ 幂等：多次执行结果相同

---

## 3. 使用方法

### 3.1 基本用法：创建 Service

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
    
    operationResult, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
        // 设置期望状态
        service.Spec.Selector = recLabels
        service.Spec.Type = corev1.ServiceTypeClusterIP
        service.Spec.Ports = []corev1.ServicePort{{
            Name:       "http",
            Port:       8080,
            TargetPort: intstr.FromInt(8080),
            Protocol:   corev1.ProtocolTCP,
        }}
        
        // 设置 Owner Reference（级联删除）
        return controllerutil.SetControllerReference(terminal, service, r.Scheme)
    })
    
    if err != nil {
        return fmt.Errorf("failed to sync service: %w", err)
    }
    
    logger.Info("service synced", "result", operationResult)
    return nil
}
```

### 3.2 创建 Deployment

```go
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
        // 设置 Labels
        deployment.Labels = recLabels
        
        // 设置 Spec
        deployment.Spec.Replicas = ptr.To(int32(1))
        deployment.Spec.Selector = &metav1.LabelSelector{
            MatchLabels: recLabels,
        }
        
        // 设置 Pod Template
        deployment.Spec.Template.Labels = recLabels
        deployment.Spec.Template.Spec.Hostname = *hostname
        deployment.Spec.Template.Spec.Containers = []corev1.Container{{
            Name:  "terminal",
            Image: terminal.Spec.Image,
            Ports: []corev1.ContainerPort{{
                ContainerPort: 8080,
            }},
            Resources: corev1.ResourceRequirements{
                Requests: corev1.ResourceList{
                    corev1.ResourceCPU:    resource.MustParse("0.01"),
                    corev1.ResourceMemory: resource.MustParse("16Mi"),
                },
                Limits: corev1.ResourceList{
                    corev1.ResourceCPU:    resource.MustParse("0.3"),
                    corev1.ResourceMemory: resource.MustParse("256Mi"),
                },
            },
        }}
        
        // 保存 hostname（如果是新创建）
        if *hostname == "" {
            *hostname = deployment.Spec.Template.Spec.Hostname
        }
        
        return controllerutil.SetControllerReference(terminal, deployment, r.Scheme)
    })
    
    return err
}
```

### 3.3 创建 Ingress

```go
func (r *TerminalReconciler) syncIngress(
    ctx context.Context,
    terminal *terminalv1.Terminal,
    hostname string,
) error {
    ingress := &networkingv1.Ingress{
        ObjectMeta: metav1.ObjectMeta{
            Name:      terminal.Name,
            Namespace: terminal.Namespace,
        },
    }
    
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, ingress, func() error {
        // 设置 Annotations
        if ingress.Annotations == nil {
            ingress.Annotations = make(map[string]string)
        }
        ingress.Annotations["nginx.ingress.kubernetes.io/rewrite-target"] = "/"
        ingress.Annotations["nginx.ingress.kubernetes.io/ssl-redirect"] = "true"
        
        // 设置 TLS
        ingress.Spec.TLS = []networkingv1.IngressTLS{{
            Hosts:      []string{hostname + ".cloud.sealos.io"},
            SecretName: "wildcard-cert",
        }}
        
        // 设置 Rules
        pathType := networkingv1.PathTypePrefix
        ingress.Spec.Rules = []networkingv1.IngressRule{{
            Host: hostname + ".cloud.sealos.io",
            IngressRuleValue: networkingv1.IngressRuleValue{
                HTTP: &networkingv1.HTTPIngressRuleValue{
                    Paths: []networkingv1.HTTPIngressPath{{
                        Path:     "/",
                        PathType: &pathType,
                        Backend: networkingv1.IngressBackend{
                            Service: &networkingv1.IngressServiceBackend{
                                Name: terminal.Status.ServiceName,
                                Port: networkingv1.ServiceBackendPort{
                                    Number: 8080,
                                },
                            },
                        },
                    }},
                },
            },
        }}
        
        return controllerutil.SetControllerReference(terminal, ingress, r.Scheme)
    })
    
    return err
}
```

### 3.4 条件性创建（根据 Spec 决定是否创建）

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) syncNodeport(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    recLabels map[string]string,
) error {
    service := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      devbox.Name + "-nodeport",
            Namespace: devbox.Namespace,
            Labels:    recLabels,
        },
    }
    
    // 如果不需要 NodePort，删除已存在的 Service
    if devbox.Spec.NetworkSpec.Type != devboxv1alpha2.NetworkTypeNodePort {
        return client.IgnoreNotFound(r.Delete(ctx, service))
    }
    
    // 需要 NodePort，创建或更新
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
        service.Spec.Selector = recLabels
        service.Spec.Type = corev1.ServiceTypeNodePort
        service.Spec.Ports = buildServicePorts(devbox)
        return controllerutil.SetControllerReference(devbox, service, r.Scheme)
    })
    
    return err
}
```

---

## 4. 与 Owner Reference 配合

### 4.1 为什么需要 Owner Reference

Owner Reference 实现了 Kubernetes 的**级联删除**机制：

```
Terminal (Owner)
  └─ Deployment (Owned)
  └─ Service (Owned)
  └─ Ingress (Owned)

删除 Terminal → 自动删除所有子资源
```

### 4.2 设置 Owner Reference

```go
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    // ... 设置 Spec ...
    
    // 设置 Owner Reference
    return controllerutil.SetControllerReference(terminal, service, r.Scheme)
})
```

**SetControllerReference 做了什么**：

1. 在 `service.OwnerReferences` 中添加 `terminal` 的引用
2. 设置 `Controller: true` 和 `BlockOwnerDeletion: true`
3. 验证 Owner 和 Owned 在同一 Namespace（Cluster-scoped 资源除外）

**生成的 OwnerReference**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: terminal-svc
  namespace: user-ns
  ownerReferences:
  - apiVersion: terminal.sealos.io/v1
    kind: Terminal
    name: my-terminal
    uid: 12345-67890
    controller: true
    blockOwnerDeletion: true
spec:
  # ...
```

### 4.3 限制

| 限制 | 说明 | 解决方案 |
|------|------|---------|
| 同 Namespace | Owner 和 Owned 必须在同一 Namespace | 使用 Label 关联跨 Namespace 资源 |
| 单一 Controller | 一个资源只能有一个 Controller Owner | 使用非 Controller 的 OwnerReference |
| Cluster-scoped | Namespace-scoped 资源不能 Own Cluster-scoped 资源 | 使用 Finalizer 手动清理 |

---

## 5. 最佳实践

### 5.1 ✅ 只在 Mutate 函数中设置期望字段

```go
// ✅ 正确：只设置你管理的字段
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    service.Spec.Type = corev1.ServiceTypeClusterIP
    service.Spec.Selector = recLabels
    service.Spec.Ports = ports
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})

// ❌ 错误：覆盖所有字段
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    *service = corev1.Service{  // 覆盖整个对象
        ObjectMeta: metav1.ObjectMeta{
            Name:      "my-service",
            Namespace: "default",
        },
        Spec: corev1.ServiceSpec{...},
    }
    return nil
})
```

### 5.2 ✅ 在 Mutate 函数中初始化 Map/Slice

```go
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, ingress, func() error {
    // 初始化 Annotations（如果为 nil）
    if ingress.Annotations == nil {
        ingress.Annotations = make(map[string]string)
    }
    ingress.Annotations["nginx.ingress.kubernetes.io/rewrite-target"] = "/"
    
    // 直接赋值 Slice（覆盖整个 Slice）
    ingress.Spec.Rules = []networkingv1.IngressRule{...}
    
    return controllerutil.SetControllerReference(owner, ingress, r.Scheme)
})
```

### 5.3 ✅ 使用返回值判断操作类型

```go
result, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    // ...
})

if err != nil {
    return err
}

switch result {
case controllerutil.OperationResultCreated:
    logger.Info("service created", "name", service.Name)
    r.Recorder.Event(owner, corev1.EventTypeNormal, "Created", "Service created")
case controllerutil.OperationResultUpdated:
    logger.Info("service updated", "name", service.Name)
case controllerutil.OperationResultNone:
    logger.V(1).Info("service unchanged", "name", service.Name)
}
```

### 5.4 ✅ 处理不可变字段

某些字段创建后不可修改（如 Service 的 ClusterIP）：

```go
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    // 只在创建时设置 ClusterIP
    if service.CreationTimestamp.IsZero() {
        service.Spec.ClusterIP = "10.96.0.100"
    }
    
    // 可变字段始终设置
    service.Spec.Type = corev1.ServiceTypeClusterIP
    service.Spec.Ports = ports
    
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})
```

### 5.5 ✅ 结合 RetryOnConflict 处理并发

```go
// 高并发场景下，CreateOrUpdate 也可能冲突
err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    service := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "my-service",
            Namespace: "default",
        },
    }
    
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
        service.Spec.Type = corev1.ServiceTypeClusterIP
        service.Spec.Ports = ports
        return controllerutil.SetControllerReference(owner, service, r.Scheme)
    })
    return err
})
```

---

## 6. 常见错误

### 6.1 ❌ 在 Mutate 函数外修改对象

```go
// ❌ 错误：在 Mutate 函数外修改
service := &corev1.Service{...}
service.Spec.Type = corev1.ServiceTypeClusterIP  // ❌ 无效

_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})

// ✅ 正确：在 Mutate 函数内修改
service := &corev1.Service{...}
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    service.Spec.Type = corev1.ServiceTypeClusterIP  // ✅ 有效
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})
```

### 6.2 ❌ 覆盖整个对象

```go
// ❌ 错误：覆盖整个对象会丢失系统字段
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    *service = corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "my-service",
            Namespace: "default",
        },
        Spec: corev1.ServiceSpec{...},
    }
    return nil  // ResourceVersion、UID 等字段丢失
})

// ✅ 正确：只修改需要的字段
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    service.Spec.Type = corev1.ServiceTypeClusterIP
    service.Spec.Ports = ports
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})
```

### 6.3 ❌ 忘记设置 Name 和 Namespace

```go
// ❌ 错误：CreateOrUpdate 需要通过 Name 和 Namespace 查找对象
service := &corev1.Service{}  // Name 和 Namespace 为空

_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    service.Spec.Type = corev1.ServiceTypeClusterIP
    return nil
})
// 错误：无法确定对象的 Key

// ✅ 正确：必须设置 Name 和 Namespace
service := &corev1.Service{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-service",
        Namespace: "default",
    },
}
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    service.Spec.Type = corev1.ServiceTypeClusterIP
    return nil
})
```

### 6.4 ❌ 在 Mutate 函数中执行耗时操作

```go
// ❌ 错误：Mutate 函数应该快速执行
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    // ❌ 不要在这里查询其他资源
    pod := &corev1.Pod{}
    r.Get(ctx, key, pod)
    
    // ❌ 不要在这里执行复杂计算
    result := heavyComputation()
    
    service.Spec.Type = corev1.ServiceTypeClusterIP
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})

// ✅ 正确：在 Mutate 函数外准备数据
pod := &corev1.Pod{}
r.Get(ctx, key, pod)
result := heavyComputation()

_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    service.Spec.Type = corev1.ServiceTypeClusterIP
    service.Annotations["result"] = result
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})
```

### 6.5 ❌ 跨 Namespace 设置 Owner Reference

```go
// ❌ 错误：Owner 和 Owned 必须在同一 Namespace
owner := &terminalv1.Terminal{
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

_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
    // 错误：cross-namespace owner references are disallowed
})

// ✅ 正确：使用 Label 关联跨 Namespace 资源
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

_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    service.Spec.Type = corev1.ServiceTypeClusterIP
    return nil  // 不设置 Owner Reference
})
```

---

## 7. 实际案例

### 7.1 案例 1：Terminal 控制器的资源同步

Terminal 控制器需要为每个 Terminal 创建 Deployment、Service、Ingress：

```go
// controllers/terminal/controllers/terminal_controller.go
func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &terminalv1.Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 推荐标签
    recLabels := label.RecommendedLabels(&label.Recommended{
        Name:      terminal.Name,
        ManagedBy: label.DefaultManagedBy,
        PartOf:    "terminal",
    })
    
    var hostname string
    
    // 同步 Deployment
    if err := r.syncDeployment(ctx, terminal, &hostname, recLabels); err != nil {
        r.recorder.Eventf(terminal, corev1.EventTypeWarning, "SyncFailed", "Failed to sync deployment: %v", err)
        return ctrl.Result{}, err
    }
    
    // 同步 Service
    if err := r.syncService(ctx, terminal, recLabels); err != nil {
        r.recorder.Eventf(terminal, corev1.EventTypeWarning, "SyncFailed", "Failed to sync service: %v", err)
        return ctrl.Result{}, err
    }
    
    // 同步 Ingress
    if err := r.syncIngress(ctx, terminal, hostname); err != nil {
        r.recorder.Eventf(terminal, corev1.EventTypeWarning, "SyncFailed", "Failed to sync ingress: %v", err)
        return ctrl.Result{}, err
    }
    
    r.recorder.Event(terminal, corev1.EventTypeNormal, "Synced", "All resources synced successfully")
    return ctrl.Result{}, nil
}
```

**优点**：
- ✅ 每个资源的同步逻辑独立
- ✅ 幂等：多次调用结果相同
- ✅ 自动级联删除

### 7.2 案例 2：User 控制器的 Namespace 创建

User 控制器为每个用户创建专属 Namespace：

```go
// controllers/user/controllers/user_controller.go
func (r *UserReconciler) syncNamespace(ctx context.Context, user *userv1.User) context.Context {
    condition := &userv1.Condition{
        Type:              userv1.ConditionTypeNamespace,
        Status:            corev1.ConditionTrue,
        LastHeartbeatTime: metav1.Now(),
        Reason:            string(userv1.ConditionTypeNamespace),
        Message:           "namespace is available",
    }
    
    nsCondition := condition.DeepCopy()
    defer func() {
        if helper.DiffCondition(condition, nsCondition) {
            r.saveCondition(user, nsCondition.DeepCopy())
        }
    }()
    
    if err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        ns := &v1.Namespace{}
        ns.Name = config.GetUsersNamespace(user.Name)
        ns.Labels = map[string]string{}
        
        _, err := controllerutil.CreateOrUpdate(ctx, r.Client, ns, func() error {
            // 设置 Labels
            if ns.Labels == nil {
                ns.Labels = make(map[string]string)
            }
            ns.Labels[userv1.UserLabelOwnerKey] = user.Name
            
            // 设置 Annotations
            if ns.Annotations == nil {
                ns.Annotations = make(map[string]string)
            }
            ns.Annotations[userv1.UserAnnotationCreatorKey] = user.Annotations[userv1.UserAnnotationCreatorKey]
            
            return nil  // 不设置 Owner Reference（跨 Namespace）
        })
        return err
    }); err != nil {
        nsCondition.Status = corev1.ConditionFalse
        nsCondition.Reason = "CreateNamespaceFailed"
        nsCondition.Message = err.Error()
    }
    
    return ctx
}
```

**分析**：
- ✅ 结合 `RetryOnConflict` 处理并发
- ✅ 使用 Label 关联 User 和 Namespace
- ✅ 更新 Condition 反馈状态

### 7.3 案例 3：Devbox 控制器的条件性资源创建

Devbox 根据 `NetworkSpec.Type` 决定是否创建 NodePort Service：

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) syncNodeport(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    recLabels map[string]string,
) error {
    service := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      devbox.Name + "-nodeport",
            Namespace: devbox.Namespace,
            Labels:    recLabels,
        },
    }
    
    // 不需要 NodePort，删除已存在的 Service
    if devbox.Spec.NetworkSpec.Type != devboxv1alpha2.NetworkTypeNodePort {
        return r.deleteNodeport(ctx, devbox, service)
    }
    
    // 需要 NodePort，创建或更新
    if _, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
        service.Spec.Selector = recLabels
        service.Spec.Type = corev1.ServiceTypeNodePort
        service.Spec.Ports = buildServicePorts(devbox)
        return controllerutil.SetControllerReference(devbox, service, r.Scheme)
    }); err != nil {
        return err
    }
    
    // 更新 Status 中的 NodePort
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
            return err
        }
        latest.Status.Network.NodePort = getNodePort(service)
        return r.Status().Update(ctx, latest)
    })
}

func (r *DevboxReconciler) deleteNodeport(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    service *corev1.Service,
) error {
    if err := r.Delete(ctx, service); err != nil {
        return client.IgnoreNotFound(err)
    }
    
    // 清空 Status 中的 NodePort
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
            return err
        }
        latest.Status.Network.NodePort = 0
        return r.Status().Update(ctx, latest)
    })
}
```

**分析**：
- ✅ 根据 Spec 动态创建/删除资源
- ✅ 同步更新 Status
- ✅ 使用 `IgnoreNotFound` 处理删除场景

---

## 8. 性能优化

### 8.1 批量创建资源

```go
// ❌ 低效：串行创建
for _, port := range ports {
    service := &corev1.Service{...}
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
        // ...
    })
}

// ✅ 高效：并行创建（如果资源独立）
var wg sync.WaitGroup
errCh := make(chan error, len(ports))

for _, port := range ports {
    wg.Add(1)
    go func(p int) {
        defer wg.Done()
        service := &corev1.Service{...}
        _, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
            // ...
        })
        if err != nil {
            errCh <- err
        }
    }(port)
}

wg.Wait()
close(errCh)

for err := range errCh {
    return err
}
```

### 8.2 避免不必要的更新

```go
// ✅ 在 Mutate 函数中检查是否需要更新
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
    needsUpdate := false
    
    // 检查 Type
    if service.Spec.Type != corev1.ServiceTypeClusterIP {
        service.Spec.Type = corev1.ServiceTypeClusterIP
        needsUpdate = true
    }
    
    // 检查 Ports
    if !reflect.DeepEqual(service.Spec.Ports, expectedPorts) {
        service.Spec.Ports = expectedPorts
        needsUpdate = true
    }
    
    if !needsUpdate {
        return nil  // 无变化，跳过更新
    }
    
    return controllerutil.SetControllerReference(owner, service, r.Scheme)
})
```

### 8.3 使用 Predicate 减少 Reconcile

```go
func (r *TerminalReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&terminalv1.Terminal{}, builder.WithPredicates(
            predicate.GenerationChangedPredicate{},  // 只在 Spec 变化时触发
        )).
        Owns(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Owns(&networkingv1.Ingress{}).
        Complete(r)
}
```

---

## 9. 总结

### 核心要点

1. ✅ **在 Mutate 函数中设置期望状态** - 只修改你管理的字段
2. ✅ **始终设置 Owner Reference** - 实现级联删除
3. ✅ **初始化 Map/Slice** - 避免 nil pointer
4. ✅ **处理不可变字段** - 检查 CreationTimestamp
5. ✅ **使用返回值** - 记录操作类型

### 适用场景

| 场景 | 是否使用 CreateOrUpdate |
|------|------------------------|
| 创建子资源 | ✅ 是 |
| 更新子资源 | ✅ 是 |
| 删除子资源 | ❌ 否（使用 Delete） |
| 更新资源状态 | ❌ 否（使用 Status().Update()） |
| 添加 Finalizer | ❌ 否（使用 Update()） |

### 与其他模式的配合

- **Owner Reference 模式**：在 Mutate 函数中设置 Owner Reference
- **RetryOnConflict 模式**：外层包裹 RetryOnConflict 处理并发
- **Pipeline 模式**：在 Pipeline 的每个步骤中同步不同的子资源

### 参考资源

- [controllerutil 包文档](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/controller/controllerutil)
- [Owner References 文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
- [Sealos Terminal Controller 实现](https://github.com/labring/sealos/blob/main/controllers/terminal/controllers/terminal_controller.go)
