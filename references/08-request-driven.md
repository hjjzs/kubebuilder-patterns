# Request-Driven 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 一次性操作、异步处理、操作审计

## 概述

Request-Driven 模式通过创建 Request 类型的 CR 对象来触发操作，Controller 监听这些请求并执行相应的业务逻辑，最后通过 Status 字段反馈执行结果。这种模式将操作请求和执行解耦，适合一次性的异步操作。

## 快速开始

### 传统 Controller 模式 vs Request-Driven 模式

```go
// ❌ 传统模式：直接修改资源触发操作
// 问题：难以追踪操作历史，无法审计

// 1. 用户修改 User 资源
user.Spec.Role = "Manager"
kubectl apply -f user.yaml

// 2. Controller 检测到变化，执行操作
func (r *UserReconciler) Reconcile(ctx context.Context, req ctrl.Request) {
    // 直接执行操作，无法知道是谁触发的
}
```

```go
// ✅ Request-Driven 模式：创建 Request CR 触发操作
// 优点：可追踪、可审计、可重试

// 1. 创建 OperationRequest
apiVersion: user.sealos.io/v1
kind: Operationrequest
metadata:
  name: grant-manager-role
spec:
  user: alice
  namespace: ns-alice
  role: Manager
  action: Grant

// 2. Controller 监听 Request，执行操作
func (r *OperationrequestReconciler) Reconcile(ctx context.Context, req ctrl.Request) {
    opReq := &Operationrequest{}
    r.Get(ctx, req.NamespacedName, opReq)
    
    // 执行操作
    if err := r.grantRole(ctx, opReq); err != nil {
        opReq.Status.Phase = "Failed"
        return ctrl.Result{}, err
    }
    
    // 标记完成
    opReq.Status.Phase = "Completed"
    r.Status().Update(ctx, opReq)
}
```

---

## 1. 核心概念

### 1.1 传统模式 vs Request-Driven 模式

| 特性 | 传统 Controller 模式 | Request-Driven 模式 |
|------|---------------------|---------------------|
| 触发方式 | 资源变更触发 | 创建 Request CR 触发 |
| 操作语义 | 声明式（期望状态） | 命令式（操作请求） |
| 生命周期 | 资源持续存在 | 请求完成后可清理 |
| 状态追踪 | 通过资源 Status | 通过 Request Status |
| 幂等性 | 内置（调谐到期望状态） | 需要显式处理 |
| 审计 | 困难 | 简单（保留 Request 记录） |

### 1.2 适用场景

- ✅ **一次性操作**：用户删除、权限授予/撤销
- ✅ **需要审计**：操作记录需要保留一段时间
- ✅ **异步处理**：操作可能需要较长时间完成
- ✅ **解耦调用方和执行方**：API 服务创建请求，Controller 执行

---

## 2. CRD 设计

### 2.1 Spec 定义（操作参数）

```go
// controllers/user/api/v1/operationrequest_types.go
type OperationrequestSpec struct {
    // Namespace is the workspace that needs to be operated
    Namespace string `json:"namespace,omitempty"`
    
    // User is the target user
    User string `json:"user,omitempty"`
    
    // Role defines the role type
    // +kubebuilder:validation:Enum=Owner;Manager;Developer
    Role RoleType `json:"role,omitempty"`
    
    // Action defines what operation to perform
    // +kubebuilder:validation:Enum=Grant;Update;Deprive
    Action ActionType `json:"action,omitempty"`
}

type RoleType string

const (
    RoleOwner     RoleType = "Owner"
    RoleManager   RoleType = "Manager"
    RoleDeveloper RoleType = "Developer"
)

type ActionType string

const (
    ActionGrant   ActionType = "Grant"   // 授予权限
    ActionUpdate  ActionType = "Update"  // 更新权限
    ActionDeprive ActionType = "Deprive" // 撤销权限
)
```

### 2.2 Status 定义（执行状态）

```go
type OperationrequestStatus struct {
    // Phase is the recently observed lifecycle phase of operationrequest
    // +kubebuilder:default:=Pending
    // +kubebuilder:validation:Enum=Pending;Processing;Completed;Failed
    Phase RequestPhase `json:"phase,omitempty"`
    
    // Message provides additional information about the current phase
    Message string `json:"message,omitempty"`
    
    // CompletionTime is the time when the request completed
    CompletionTime *metav1.Time `json:"completionTime,omitempty"`
}

type RequestPhase string

const (
    RequestPending    RequestPhase = "Pending"
    RequestProcessing RequestPhase = "Processing"
    RequestCompleted  RequestPhase = "Completed"
    RequestFailed     RequestPhase = "Failed"
)
```

---

## 3. Controller 实现

### 3.1 基本流程

```go
// controllers/user/controllers/operationrequest_controller.go
func (r *OperationrequestReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    // 1. 获取 Request
    opReq := &userv1.Operationrequest{}
    if err := r.Get(ctx, req.NamespacedName, opReq); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. 已完成的请求不再处理
    if opReq.Status.Phase == userv1.RequestCompleted || 
       opReq.Status.Phase == userv1.RequestFailed {
        return ctrl.Result{}, nil
    }
    
    // 3. 更新为 Processing
    if opReq.Status.Phase != userv1.RequestProcessing {
        opReq.Status.Phase = userv1.RequestProcessing
        if err := r.Status().Update(ctx, opReq); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 4. 执行操作
    if err := r.executeOperation(ctx, opReq); err != nil {
        logger.Error(err, "failed to execute operation")
        
        opReq.Status.Phase = userv1.RequestFailed
        opReq.Status.Message = err.Error()
        r.Status().Update(ctx, opReq)
        
        // 记录事件
        r.Recorder.Event(opReq, corev1.EventTypeWarning, "OperationFailed", err.Error())
        return ctrl.Result{}, err
    }
    
    // 5. 标记完成
    opReq.Status.Phase = userv1.RequestCompleted
    opReq.Status.CompletionTime = &metav1.Time{Time: time.Now()}
    opReq.Status.Message = "Operation completed successfully"
    
    if err := r.Status().Update(ctx, opReq); err != nil {
        return ctrl.Result{}, err
    }
    
    // 记录事件
    r.Recorder.Event(opReq, corev1.EventTypeNormal, "OperationCompleted", "Operation completed successfully")
    
    return ctrl.Result{}, nil
}
```

### 3.2 操作执行逻辑

```go
func (r *OperationrequestReconciler) executeOperation(
    ctx context.Context,
    opReq *userv1.Operationrequest,
) error {
    switch opReq.Spec.Action {
    case userv1.ActionGrant:
        return r.grantRole(ctx, opReq)
    case userv1.ActionUpdate:
        return r.updateRole(ctx, opReq)
    case userv1.ActionDeprive:
        return r.depriveRole(ctx, opReq)
    default:
        return fmt.Errorf("unknown action: %s", opReq.Spec.Action)
    }
}

func (r *OperationrequestReconciler) grantRole(
    ctx context.Context,
    opReq *userv1.Operationrequest,
) error {
    // 创建 RoleBinding
    rb := &rbacv1.RoleBinding{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("%s-%s", opReq.Spec.User, opReq.Spec.Role),
            Namespace: opReq.Spec.Namespace,
        },
    }
    
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, rb, func() error {
        rb.RoleRef = rbacv1.RoleRef{
            APIGroup: "rbac.authorization.k8s.io",
            Kind:     "Role",
            Name:     string(opReq.Spec.Role),
        }
        rb.Subjects = []rbacv1.Subject{{
            Kind: "User",
            Name: opReq.Spec.User,
        }}
        return nil
    })
    
    return err
}
```

---

## 4. 生命周期管理

### 4.1 自动清理

```go
// 方式 1: TTL Controller（推荐）
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
type Operationrequest struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   OperationrequestSpec   `json:"spec,omitempty"`
    Status OperationrequestStatus `json:"status,omitempty"`
}

// 在 Controller 中实现 TTL 清理
func (r *OperationrequestReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    opReq := &userv1.Operationrequest{}
    if err := r.Get(ctx, req.NamespacedName, opReq); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 已完成的请求，1 小时后删除
    if opReq.Status.Phase == userv1.RequestCompleted && 
       opReq.Status.CompletionTime != nil {
        if time.Since(opReq.Status.CompletionTime.Time) > time.Hour {
            if err := r.Delete(ctx, opReq); err != nil {
                return ctrl.Result{}, client.IgnoreNotFound(err)
            }
            return ctrl.Result{}, nil
        }
        
        // 重新入队，1 小时后再检查
        return ctrl.Result{RequeueAfter: time.Hour}, nil
    }
    
    // 执行操作
    // ...
}
```

### 4.2 手动清理

```go
// 方式 2: 定期清理脚本
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-operation-requests
spec:
  schedule: "0 * * * *"  # 每小时执行
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # 删除 1 小时前完成的请求
              kubectl delete operationrequests \
                --field-selector status.phase=Completed \
                --all-namespaces \
                --older-than=1h
          restartPolicy: OnFailure
```

---

## 5. 实际案例

### 案例 1：User 控制器的权限管理

```go
// controllers/user/controllers/operationrequest_controller.go
type OperationrequestReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Logger   logr.Logger
    Recorder record.EventRecorder
}

func (r *OperationrequestReconciler) SetupWithManager(mgr ctrl.Manager, opts ratelimiter.Options) error {
    r.Logger = ctrl.Log.WithName("operationrequest-controller")
    r.Recorder = mgr.GetEventRecorderFor("operationrequest-controller")
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&userv1.Operationrequest{}, builder.WithPredicates(
            namespaceOnlyPredicate(config.GetUserSystemNamespace()),
        )).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: ratelimiter.GetConcurrent(opts),
            RateLimiter:             ratelimiter.GetRateLimiter(opts),
        }).
        Complete(r)
}
```

### 案例 2：DeleteRequest（用户删除请求）

```go
// controllers/user/api/v1/deleterequest_types.go
type DeleterequestSpec struct {
    // User is the user to be deleted
    User string `json:"user"`
    
    // DeleteDependents specifies whether to delete dependent resources
    // +kubebuilder:default:=true
    DeleteDependents bool `json:"deleteDependents,omitempty"`
}

type DeleterequestStatus struct {
    Phase RequestPhase `json:"phase,omitempty"`
}

// Controller 实现
func (r *DeleterequestReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    delReq := &userv1.Deleterequest{}
    if err := r.Get(ctx, req.NamespacedName, delReq); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 已完成
    if delReq.Status.Phase == userv1.RequestCompleted {
        return ctrl.Result{}, nil
    }
    
    // 更新为 Processing
    delReq.Status.Phase = userv1.RequestProcessing
    if err := r.Status().Update(ctx, delReq); err != nil {
        return ctrl.Result{}, err
    }
    
    // 删除 User
    user := &userv1.User{
        ObjectMeta: metav1.ObjectMeta{
            Name: delReq.Spec.User,
        },
    }
    if err := r.Delete(ctx, user); err != nil && !apierrors.IsNotFound(err) {
        delReq.Status.Phase = userv1.RequestFailed
        r.Status().Update(ctx, delReq)
        return ctrl.Result{}, err
    }
    
    // 删除依赖资源
    if delReq.Spec.DeleteDependents {
        if err := r.deleteDependentResources(ctx, delReq.Spec.User); err != nil {
            delReq.Status.Phase = userv1.RequestFailed
            r.Status().Update(ctx, delReq)
            return ctrl.Result{}, err
        }
    }
    
    // 标记完成
    delReq.Status.Phase = userv1.RequestCompleted
    return ctrl.Result{}, r.Status().Update(ctx, delReq)
}
```

---

## 6. 最佳实践

### 6.1 ✅ 幂等性处理

```go
// ✅ 正确：检查操作是否已执行
func (r *OperationrequestReconciler) grantRole(ctx context.Context, opReq *userv1.Operationrequest) error {
    rb := &rbacv1.RoleBinding{}
    key := client.ObjectKey{
        Name:      fmt.Sprintf("%s-%s", opReq.Spec.User, opReq.Spec.Role),
        Namespace: opReq.Spec.Namespace,
    }
    
    // 检查是否已存在
    if err := r.Get(ctx, key, rb); err == nil {
        // 已存在，检查是否匹配
        if rb.RoleRef.Name == string(opReq.Spec.Role) &&
           len(rb.Subjects) > 0 &&
           rb.Subjects[0].Name == opReq.Spec.User {
            return nil  // 已经授予，幂等
        }
    }
    
    // 创建或更新
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, rb, func() error {
        // ...
    })
    return err
}
```

### 6.2 ✅ 错误重试

```go
// ✅ 正确：失败后重新入队
func (r *OperationrequestReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    opReq := &userv1.Operationrequest{}
    if err := r.Get(ctx, req.NamespacedName, opReq); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 执行操作
    if err := r.executeOperation(ctx, opReq); err != nil {
        // 临时错误，重试
        if isTemporaryError(err) {
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
        
        // 永久错误，标记失败
        opReq.Status.Phase = userv1.RequestFailed
        opReq.Status.Message = err.Error()
        r.Status().Update(ctx, opReq)
        return ctrl.Result{}, nil
    }
    
    // 成功
    opReq.Status.Phase = userv1.RequestCompleted
    return ctrl.Result{}, r.Status().Update(ctx, opReq)
}

func isTemporaryError(err error) bool {
    // 网络错误、API Server 限流等
    return apierrors.IsServerTimeout(err) ||
           apierrors.IsTimeout(err) ||
           apierrors.IsTooManyRequests(err)
}
```

### 6.3 ✅ 审计日志

```go
// ✅ 正确：记录详细的审计信息
func (r *OperationrequestReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    opReq := &userv1.Operationrequest{}
    if err := r.Get(ctx, req.NamespacedName, opReq); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 记录操作开始
    r.Logger.Info("operation started",
        "request", opReq.Name,
        "action", opReq.Spec.Action,
        "user", opReq.Spec.User,
        "role", opReq.Spec.Role,
        "namespace", opReq.Spec.Namespace,
        "creator", opReq.Annotations["creator"],
    )
    
    // 执行操作
    startTime := time.Now()
    err := r.executeOperation(ctx, opReq)
    duration := time.Since(startTime)
    
    if err != nil {
        // 记录失败
        r.Logger.Error(err, "operation failed",
            "request", opReq.Name,
            "duration", duration,
        )
        r.Recorder.Event(opReq, corev1.EventTypeWarning, "OperationFailed", err.Error())
    } else {
        // 记录成功
        r.Logger.Info("operation completed",
            "request", opReq.Name,
            "duration", duration,
        )
        r.Recorder.Event(opReq, corev1.EventTypeNormal, "OperationCompleted", "Operation completed successfully")
    }
    
    return ctrl.Result{}, nil
}
```

---

## 7. 总结

### 核心要点

1. ✅ **命令式操作** - 适合一次性操作
2. ✅ **状态追踪** - 通过 Status 反馈执行结果
3. ✅ **审计友好** - 保留操作记录
4. ✅ **幂等处理** - 确保重复执行安全
5. ✅ **自动清理** - 避免 Request 堆积

### 适用场景

| 场景 | 是否使用 Request-Driven |
|------|----------------------|
| 用户删除 | ✅ 是 |
| 权限授予/撤销 | ✅ 是 |
| 一次性迁移 | ✅ 是 |
| 资源同步 | ❌ 否（使用传统 Controller） |
| 持续调谐 | ❌ 否（使用传统 Controller） |

### 与其他模式的配合

- **RetryOnConflict**: 更新 Request Status 时使用
- **Event Recording**: 记录操作审计日志
- **TTL/Expiration**: 自动清理完成的 Request

### 参考资源

- [详细文档](../kubebuilder-request-driven-pattern.md)
- [Sealos User Controller 实现](https://github.com/labring/sealos/blob/main/controllers/user/controllers/operationrequest_controller.go)
