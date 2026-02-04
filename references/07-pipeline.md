# Pipeline 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 多步骤编排、复杂 Reconcile 逻辑

## 概述

Pipeline 模式通过将复杂的 Reconcile 逻辑拆分为多个独立的步骤（Pipeline Stages），使代码更清晰、易于测试和维护。每个步骤专注于单一职责，步骤之间通过 Context 传递状态。

## 快速开始

### 传统方式 vs Pipeline 模式

```go
// ❌ 传统方式：所有逻辑堆在一起
func (r *UserReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    user := &userv1.User{}
    if err := r.Get(ctx, req.NamespacedName, user); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 初始化状态
    if user.Status.Phase == "" {
        user.Status.Phase = userv1.UserPhasePending
    }
    
    // 创建 Namespace
    ns := &corev1.Namespace{}
    ns.Name = config.GetUsersNamespace(user.Name)
    if _, err := controllerutil.CreateOrUpdate(ctx, r.Client, ns, func() error {
        // ...
    }); err != nil {
        user.Status.Conditions = append(user.Status.Conditions, errorCondition)
        return ctrl.Result{}, err
    }
    
    // 创建 ServiceAccount
    sa := &corev1.ServiceAccount{}
    // ... 更多逻辑 ...
    
    // 创建 Role
    role := &rbacv1.Role{}
    // ... 更多逻辑 ...
    
    // 更新状态
    return ctrl.Result{}, r.Status().Update(ctx, user)
}
```

```go
// ✅ Pipeline 模式：清晰的步骤划分
func (r *UserReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    user := &userv1.User{}
    if err := r.Get(ctx, req.NamespacedName, user); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 定义 Pipeline
    pipelines := []func(ctx context.Context, user *userv1.User) context.Context{
        r.initStatus,           // 步骤 1: 初始化状态
        r.syncNamespace,        // 步骤 2: 同步 Namespace
        r.syncServiceAccount,   // 步骤 3: 同步 ServiceAccount
        r.syncRole,             // 步骤 4: 同步 Role
        r.syncRoleBinding,      // 步骤 5: 同步 RoleBinding
        r.syncFinalStatus,      // 步骤 6: 更新最终状态
    }
    
    // 执行 Pipeline
    for _, fn := range pipelines {
        ctx = fn(ctx, user)
    }
    
    // 批量更新状态
    return r.updateStatus(ctx, user)
}
```

---

## 实现方式

### 方式一：Context 传递状态

```go
// controllers/user/controllers/user_controller.go
type ctxKey string

const (
    ctxKeyConditions ctxKey = "conditions"
)

func (r *UserReconciler) syncNamespace(ctx context.Context, user *userv1.User) context.Context {
    condition := &userv1.Condition{
        Type:   userv1.ConditionTypeNamespace,
        Status: corev1.ConditionTrue,
        Reason: "NamespaceReady",
    }
    
    // 执行同步逻辑
    if err := r.createNamespace(ctx, user); err != nil {
        condition.Status = corev1.ConditionFalse
        condition.Reason = "CreateFailed"
        condition.Message = err.Error()
    }
    
    // 保存 Condition 到 Context
    conditions := getConditions(ctx)
    conditions = append(conditions, condition)
    return context.WithValue(ctx, ctxKeyConditions, conditions)
}

func (r *UserReconciler) syncFinalStatus(ctx context.Context, user *userv1.User) context.Context {
    // 从 Context 获取所有 Condition
    conditions := getConditions(ctx)
    user.Status.Conditions = conditions
    
    // 根据 Conditions 推导 Phase
    allReady := true
    for _, cond := range conditions {
        if cond.Status != corev1.ConditionTrue {
            allReady = false
            break
        }
    }
    
    if allReady {
        user.Status.Phase = userv1.UserPhaseActive
    } else {
        user.Status.Phase = userv1.UserPhasePending
    }
    
    return ctx
}

func getConditions(ctx context.Context) []userv1.Condition {
    if conditions, ok := ctx.Value(ctxKeyConditions).([]userv1.Condition); ok {
        return conditions
    }
    return []userv1.Condition{}
}
```

### 方式二：直接修改对象

```go
func (r *DevboxReconciler) reconcile(ctx context.Context, devbox *devboxv1alpha2.Devbox) error {
    pipelines := []func(context.Context, *devboxv1alpha2.Devbox) error{
        r.syncSecret,
        r.syncNetwork,
        r.syncPod,
        r.syncDevboxPhase,
    }
    
    for _, fn := range pipelines {
        if err := fn(ctx, devbox); err != nil {
            return err
        }
    }
    
    return nil
}

func (r *DevboxReconciler) syncSecret(ctx context.Context, devbox *devboxv1alpha2.Devbox) error {
    // 直接修改 devbox.Status
    devbox.Status.Network.Type = devbox.Spec.NetworkSpec.Type
    
    // 执行同步逻辑
    return r.createSecret(ctx, devbox)
}
```

---

## 优点

1. **代码清晰**：每个步骤职责单一，易于理解
2. **易于测试**：可以单独测试每个步骤
3. **易于维护**：添加/删除步骤不影响其他步骤
4. **易于调试**：可以在步骤之间添加日志
5. **可复用**：步骤可以在不同的 Pipeline 中复用

---

## 实际案例

### 案例 1：User 控制器的完整 Pipeline

```go
// controllers/user/controllers/user_controller.go
func (r *UserReconciler) reconcile(ctx context.Context, obj client.Object) (ctrl.Result, error) {
    user := obj.(*userv1.User)
    
    pipelines := []func(ctx context.Context, user *userv1.User) context.Context{
        r.initStatus,
        r.syncNamespace,
        r.syncServiceAccount,
        r.syncRole,
        r.syncRoleBinding,
        r.syncFinalStatus,
    }
    
    for _, fn := range pipelines {
        ctx = fn(ctx, user)
    }
    
    return r.updateStatus(ctx, user)
}

func (r *UserReconciler) initStatus(ctx context.Context, user *userv1.User) context.Context {
    if user.Status.Phase == "" {
        user.Status.Phase = userv1.UserPhasePending
    }
    if user.Status.Conditions == nil {
        user.Status.Conditions = []userv1.Condition{}
    }
    return ctx
}

func (r *UserReconciler) syncNamespace(ctx context.Context, user *userv1.User) context.Context {
    condition := &userv1.Condition{
        Type:              userv1.ConditionTypeNamespace,
        Status:            corev1.ConditionTrue,
        LastHeartbeatTime: metav1.Now(),
        Reason:            "NamespaceReady",
        Message:           "namespace is available",
    }
    
    defer func() {
        user.Status.Conditions = updateCondition(user.Status.Conditions, condition)
    }()
    
    if err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        ns := &corev1.Namespace{}
        ns.Name = config.GetUsersNamespace(user.Name)
        
        _, err := controllerutil.CreateOrUpdate(ctx, r.Client, ns, func() error {
            if ns.Labels == nil {
                ns.Labels = make(map[string]string)
            }
            ns.Labels[userv1.UserLabelOwnerKey] = user.Name
            return nil
        })
        return err
    }); err != nil {
        condition.Status = corev1.ConditionFalse
        condition.Reason = "CreateNamespaceFailed"
        condition.Message = err.Error()
    }
    
    return ctx
}

func updateCondition(conditions []userv1.Condition, newCond *userv1.Condition) []userv1.Condition {
    for i, c := range conditions {
        if c.Type == newCond.Type {
            conditions[i] = *newCond
            return conditions
        }
    }
    return append(conditions, *newCond)
}
```

### 案例 2：Devbox 控制器的网络同步 Pipeline

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) syncNetwork(ctx context.Context, devbox *devboxv1alpha2.Devbox, recLabels map[string]string) error {
    pipeline := []func(context.Context, *devboxv1alpha2.Devbox, map[string]string) error{
        r.syncCommon,      // Headless Service
        r.syncTailnet,     // Tailscale
        r.syncSSHGate,     // SSH Gateway
        r.syncNodeport,    // NodePort Service
    }
    
    for _, fn := range pipeline {
        if err := fn(ctx, devbox, recLabels); err != nil && !apierrors.IsNotFound(err) {
            return err
        }
    }
    
    // 更新网络状态
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
            return err
        }
        latest.Status.Network.Type = devbox.Spec.NetworkSpec.Type
        return r.Status().Update(ctx, latest)
    })
}
```

---

## 最佳实践

### 1. ✅ 步骤职责单一

```go
// ✅ 正确：每个步骤只做一件事
func (r *UserReconciler) syncNamespace(ctx context.Context, user *userv1.User) context.Context {
    // 只负责 Namespace 的创建和更新
    return ctx
}

func (r *UserReconciler) syncServiceAccount(ctx context.Context, user *userv1.User) context.Context {
    // 只负责 ServiceAccount 的创建和更新
    return ctx
}

// ❌ 错误：一个步骤做多件事
func (r *UserReconciler) syncAll(ctx context.Context, user *userv1.User) context.Context {
    // 创建 Namespace
    // 创建 ServiceAccount
    // 创建 Role
    // ... 所有逻辑都在一个函数中
    return ctx
}
```

### 2. ✅ 使用 defer 更新 Condition

```go
func (r *UserReconciler) syncNamespace(ctx context.Context, user *userv1.User) context.Context {
    condition := &userv1.Condition{
        Type:   userv1.ConditionTypeNamespace,
        Status: corev1.ConditionTrue,
    }
    
    // 使用 defer 确保 Condition 总是被更新
    defer func() {
        user.Status.Conditions = updateCondition(user.Status.Conditions, condition)
    }()
    
    // 执行逻辑
    if err := r.createNamespace(ctx, user); err != nil {
        condition.Status = corev1.ConditionFalse
        condition.Message = err.Error()
        return ctx
    }
    
    return ctx
}
```

### 3. ✅ 步骤之间独立

```go
// ✅ 正确：步骤之间独立，一个失败不影响其他
pipelines := []func(ctx context.Context, user *userv1.User) context.Context{
    r.syncNamespace,      // 失败不影响后续步骤
    r.syncServiceAccount, // 继续执行
    r.syncRole,
}

for _, fn := range pipelines {
    ctx = fn(ctx, user)  // 每个步骤都执行
}
```

### 4. ✅ 批量更新状态

```go
// ✅ 正确：所有步骤执行完后，一次性更新状态
func (r *UserReconciler) reconcile(ctx context.Context, user *userv1.User) (ctrl.Result, error) {
    // 执行所有步骤
    for _, fn := range pipelines {
        ctx = fn(ctx, user)
    }
    
    // 批量更新状态（只调用一次 API）
    return r.updateStatus(ctx, user)
}

func (r *UserReconciler) updateStatus(ctx context.Context, user *userv1.User) (ctrl.Result, error) {
    return ctrl.Result{}, retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &userv1.User{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(user), latest); err != nil {
            return err
        }
        latest.Status = user.Status
        return r.Status().Update(ctx, latest)
    })
}
```

---

## 总结

### 核心要点

1. ✅ **步骤职责单一** - 每个步骤只做一件事
2. ✅ **步骤之间独立** - 一个失败不影响其他
3. ✅ **使用 Context 传递状态** - 或直接修改对象
4. ✅ **批量更新状态** - 减少 API 调用
5. ✅ **使用 defer 更新 Condition** - 确保状态总是被更新

### 适用场景

| 场景 | 是否使用 Pipeline |
|------|-----------------|
| 多步骤同步逻辑 | ✅ 是 |
| 复杂的 Reconcile | ✅ 是 |
| 需要维护 Conditions | ✅ 是 |
| 简单的单步操作 | ❌ 否 |

### 与其他模式的配合

- **RetryOnConflict**: 在最后批量更新状态时使用
- **CreateOrUpdate**: 在每个步骤中创建/更新子资源
- **Event Recording**: 在步骤失败时记录事件

### 参考资源

- [Sealos User Controller 实现](https://github.com/labring/sealos/blob/main/controllers/user/controllers/user_controller.go)
- [Sealos Devbox Controller 实现](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
