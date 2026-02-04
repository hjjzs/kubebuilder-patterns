# Phase Derivation 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 状态聚合、Phase 推导、复杂状态管理

## 概述

Phase Derivation 模式通过分析多个信号源（Pod 状态、Commit 记录、网络状态等）来推导资源的整体 Phase，提供统一的状态视图。

## 快速开始

```go
// Devbox 的 Phase 由多个因素决定
func DerivePhase(
    state State,           // Spec 中的期望状态
    podStatus PodStatus,   // Pod 的实际状态
    commit *CommitRecord,  // 最新的 Commit 记录
) Phase {
    switch state {
    case StateRunning:
        if podStatus == PodRunning {
            return PhaseRunning
        }
        return PhasePending
        
    case StateStopped:
        if commit != nil && commit.Status == CommitStatusCommitting {
            return PhaseStopping  // 正在提交，还未完全停止
        }
        return PhaseStopped
        
    case StateShutdown:
        return PhaseShutdown
        
    default:
        return PhaseUnknown
    }
}
```

---

## 1. 核心概念

### 1.1 Spec vs Status vs Phase

| 字段 | 含义 | 设置者 | 示例 |
|------|------|--------|------|
| **Spec.State** | 用户期望状态 | 用户 | Running, Stopped |
| **Status.PodPhase** | Pod 实际状态 | Kubelet | Pending, Running, Failed |
| **Status.Phase** | 资源整体状态 | Controller | Pending, Running, Stopping, Stopped |

### 1.2 状态推导流程

```
输入信号
├─ Spec.State (用户期望)
├─ Pod.Status.Phase (Pod 状态)
├─ Commit.Status (提交状态)
└─ Network.Status (网络状态)
        │
        ▼
   ┌─────────────┐
   │ DerivePhase │
   └─────────────┘
        │
        ▼
  Status.Phase (整体状态)
```

---

## 2. 实现方式

### 2.1 基本实现

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) syncDevboxPhase(
    ctx context.Context,
    devbox *devboxv1alpha2.Devbox,
    recLabels map[string]string,
) error {
    // 1. 收集所有信号
    podList := &corev1.PodList{}
    if err := r.List(ctx, podList,
        client.InNamespace(devbox.Namespace),
        client.MatchingLabels(recLabels),
    ); err != nil {
        return err
    }
    
    // 2. 分析 Pod 状态
    podStatus := AnalyzePodStatus(podList)
    
    // 3. 获取最新 Commit 记录
    latestCommit := GetLatestCommitRecord(
        devbox.Status.CommitRecords,
        devbox.Status.ContentID,
    )
    
    // 4. 推导 Phase
    newPhase := DerivePhase(devbox.Spec.State, podStatus, latestCommit)
    
    // 5. 更新 Status（如果变化）
    if devbox.Status.Phase == newPhase {
        return nil
    }
    
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &devboxv1alpha2.Devbox{}
        if err := r.Get(ctx, client.ObjectKeyFromObject(devbox), latest); err != nil {
            return err
        }
        latest.Status.Phase = newPhase
        return r.Status().Update(ctx, latest)
    })
}
```

### 2.2 Pod 状态分析

```go
type PodStatus int

const (
    PodNotFound PodStatus = iota
    PodPending
    PodRunning
    PodFailed
    PodSucceeded
)

func AnalyzePodStatus(podList *corev1.PodList) PodStatus {
    if len(podList.Items) == 0 {
        return PodNotFound
    }
    
    pod := &podList.Items[0]
    
    switch pod.Status.Phase {
    case corev1.PodPending:
        return PodPending
    case corev1.PodRunning:
        // 检查容器状态
        if len(pod.Status.ContainerStatuses) > 0 {
            cs := pod.Status.ContainerStatuses[0]
            if cs.State.Running != nil {
                return PodRunning
            }
        }
        return PodPending
    case corev1.PodFailed:
        return PodFailed
    case corev1.PodSucceeded:
        return PodSucceeded
    default:
        return PodNotFound
    }
}
```

### 2.3 Phase 推导逻辑

```go
func DerivePhase(
    state devboxv1alpha2.DevboxState,
    podStatus PodStatus,
    commit *devboxv1alpha2.CommitRecord,
) devboxv1alpha2.DevboxPhase {
    // 状态映射表
    switch state {
    case devboxv1alpha2.DevboxStateRunning:
        switch podStatus {
        case PodRunning:
            return devboxv1alpha2.DevboxPhaseRunning
        case PodPending:
            return devboxv1alpha2.DevboxPhasePending
        case PodFailed:
            return devboxv1alpha2.DevboxPhaseError
        default:
            return devboxv1alpha2.DevboxPhasePending
        }
        
    case devboxv1alpha2.DevboxStateStopped:
        // 检查是否正在提交
        if commit != nil && commit.Status == devboxv1alpha2.CommitStatusCommitting {
            return devboxv1alpha2.DevboxPhaseStopping
        }
        return devboxv1alpha2.DevboxPhaseStopped
        
    case devboxv1alpha2.DevboxStateShutdown:
        return devboxv1alpha2.DevboxPhaseShutdown
        
    default:
        return devboxv1alpha2.DevboxPhaseUnknown
    }
}
```

---

## 3. 状态映射表

### 3.1 Devbox 状态映射

| Spec.State | Pod Status | Commit Status | → Phase |
|-----------|-----------|--------------|---------|
| Running | Running | - | Running |
| Running | Pending | - | Pending |
| Running | Failed | - | Error |
| Stopped | - | Committing | Stopping |
| Stopped | - | Completed | Stopped |
| Shutdown | - | - | Shutdown |

### 3.2 User 状态映射

| Namespace | ServiceAccount | Role | RoleBinding | → Phase |
|-----------|---------------|------|------------|---------|
| ✅ | ✅ | ✅ | ✅ | Active |
| ✅ | ✅ | ✅ | ❌ | Pending |
| ✅ | ❌ | - | - | Pending |
| ❌ | - | - | - | Pending |

---

## 4. 实际案例

### 案例 1：Devbox Phase 推导

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    recLabels := map[string]string{
        "app.kubernetes.io/name": devbox.Name,
    }
    
    // Pipeline: 依次同步各个组件
    if err := r.syncSecret(ctx, devbox, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    
    if err := r.syncNetwork(ctx, devbox, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    
    if err := r.syncPod(ctx, devbox, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    
    // 最后推导 Phase
    if err := r.syncDevboxPhase(ctx, devbox, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

### 案例 2：User Phase 推导

```go
// controllers/user/controllers/user_controller.go
func (r *UserReconciler) syncFinalStatus(ctx context.Context, user *userv1.User) context.Context {
    // 从 Context 获取所有 Condition
    conditions := getConditions(ctx)
    user.Status.Conditions = conditions
    
    // 推导 Phase：所有 Condition 都为 True 才是 Active
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
```

---

## 5. 最佳实践

### 5.1 ✅ 使用状态映射表

```go
// ✅ 正确：清晰的状态映射表
var phaseMatrix = map[State]map[PodStatus]Phase{
    StateRunning: {
        PodRunning:  PhaseRunning,
        PodPending:  PhasePending,
        PodFailed:   PhaseError,
    },
    StateStopped: {
        PodNotFound: PhaseStopped,
    },
}

func DerivePhase(state State, podStatus PodStatus) Phase {
    if phases, ok := phaseMatrix[state]; ok {
        if phase, ok := phases[podStatus]; ok {
            return phase
        }
    }
    return PhaseUnknown
}
```

### 5.2 ✅ 记录状态转换

```go
// ✅ 正确：记录 Phase 变化
func (r *DevboxReconciler) syncDevboxPhase(ctx context.Context, devbox *Devbox) error {
    oldPhase := devbox.Status.Phase
    newPhase := DerivePhase(devbox.Spec.State, podStatus, commit)
    
    if oldPhase != newPhase {
        logger.Info("phase changed",
            "devbox", devbox.Name,
            "oldPhase", oldPhase,
            "newPhase", newPhase,
        )
        
        r.Recorder.Eventf(devbox, corev1.EventTypeNormal, "PhaseChanged",
            "Phase changed from %s to %s", oldPhase, newPhase)
    }
    
    // 更新 Status
    // ...
}
```

### 5.3 ✅ 处理边界情况

```go
// ✅ 正确：处理所有可能的情况
func DerivePhase(state State, podStatus PodStatus, commit *CommitRecord) Phase {
    // 特殊情况：正在提交
    if commit != nil && commit.Status == CommitStatusCommitting {
        return PhaseStopping
    }
    
    // 特殊情况：Pod 失败
    if podStatus == PodFailed {
        return PhaseError
    }
    
    // 正常情况
    switch state {
    case StateRunning:
        if podStatus == PodRunning {
            return PhaseRunning
        }
        return PhasePending
    case StateStopped:
        return PhaseStopped
    default:
        return PhaseUnknown
    }
}
```

---

## 6. 总结

### 核心要点

1. ✅ **多信号聚合** - 综合多个状态源
2. ✅ **清晰的映射表** - 明确的状态转换规则
3. ✅ **记录状态变化** - 便于调试和审计
4. ✅ **处理边界情况** - 考虑所有可能的组合
5. ✅ **最后推导 Phase** - 在 Pipeline 最后执行

### 适用场景

| 场景 | 是否使用 Phase Derivation |
|------|-------------------------|
| 资源有多个子资源 | ✅ 是 |
| 状态由多个因素决定 | ✅ 是 |
| 需要统一的状态视图 | ✅ 是 |
| 简单的状态同步 | ❌ 否 |

### 与其他模式的配合

- **Pipeline**: 在 Pipeline 最后推导 Phase
- **RetryOnConflict**: 更新 Phase 时使用
- **Custom Predicate**: Phase 变化时触发 Reconcile

### 参考资源

- [Sealos Devbox Controller 实现](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
