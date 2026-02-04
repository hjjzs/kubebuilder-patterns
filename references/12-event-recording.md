# Event Recording 模式

**复杂度**: 🟢 简单  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 操作审计、故障诊断、用户通知

## 概述

Event Recording 模式通过 Kubernetes Event API 记录控制器的重要操作和错误信息，为用户提供可观测性，便于故障诊断和审计。

## 快速开始

```go
type TerminalReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    recorder record.EventRecorder  // Event Recorder
}

func (r *TerminalReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // 初始化 Recorder
    r.recorder = mgr.GetEventRecorderFor("terminal-controller")
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&terminalv1.Terminal{}).
        Complete(r)
}

func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &terminalv1.Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 记录正常事件
    if err := r.syncDeployment(ctx, terminal); err != nil {
        // 记录错误事件
        r.recorder.Eventf(terminal, corev1.EventTypeWarning, 
            "SyncFailed", "Failed to sync deployment: %v", err)
        return ctrl.Result{}, err
    }
    
    // 记录成功事件
    r.recorder.Event(terminal, corev1.EventTypeNormal, 
        "Synced", "Terminal synced successfully")
    
    return ctrl.Result{}, nil
}
```

---

## 1. Event API

### 1.1 Event 类型

```go
const (
    EventTypeNormal  string = "Normal"   // 正常事件
    EventTypeWarning string = "Warning"  // 警告事件
)
```

### 1.2 Recorder 方法

```go
type EventRecorder interface {
    // Event: 简单事件
    Event(object runtime.Object, eventtype, reason, message string)
    
    // Eventf: 格式化事件
    Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})
    
    // AnnotatedEventf: 带注解的事件
    AnnotatedEventf(object runtime.Object, annotations map[string]string, 
        eventtype, reason, messageFmt string, args ...interface{})
}
```

---

## 2. 使用方法

### 2.1 记录成功操作

```go
// controllers/terminal/controllers/terminal_controller.go
func (r *TerminalReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    terminal := &terminalv1.Terminal{}
    if err := r.Get(ctx, req.NamespacedName, terminal); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 同步 Deployment
    if err := r.syncDeployment(ctx, terminal, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    r.recorder.Event(terminal, corev1.EventTypeNormal, "DeploymentSynced", "Deployment synced successfully")
    
    // 同步 Service
    if err := r.syncService(ctx, terminal, recLabels); err != nil {
        return ctrl.Result{}, err
    }
    r.recorder.Event(terminal, corev1.EventTypeNormal, "ServiceSynced", "Service synced successfully")
    
    // 同步 Ingress
    if err := r.syncIngress(ctx, terminal, hostname); err != nil {
        return ctrl.Result{}, err
    }
    r.recorder.Event(terminal, corev1.EventTypeNormal, "IngressSynced", "Ingress synced successfully")
    
    return ctrl.Result{}, nil
}
```

### 2.2 记录错误

```go
// controllers/devbox/internal/controller/devbox_controller.go
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    devbox := &devboxv1alpha2.Devbox{}
    if err := r.Get(ctx, req.NamespacedName, devbox); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 创建 Pod
    if err := r.createPod(ctx, devbox); err != nil {
        r.Recorder.Eventf(devbox, corev1.EventTypeWarning,
            "CreatePodFailed", "Failed to create pod: %v", err)
        return ctrl.Result{}, err
    }
    
    r.Recorder.Event(devbox, corev1.EventTypeNormal,
        "PodCreated", "Pod created successfully")
    
    return ctrl.Result{}, nil
}
```

### 2.3 记录状态变化

```go
// controllers/devbox/internal/controller/event_handler.go
func (h *EventHandler) handleDevboxStateChange(ctx context.Context, event *corev1.Event) error {
    devbox := &devboxv1alpha2.Devbox{}
    if err := h.Client.Get(ctx, types.NamespacedName{
        Namespace: event.Namespace,
        Name:      event.InvolvedObject.Name,
    }, devbox); err != nil {
        return err
    }
    
    currentState := devbox.Status.State
    targetState := devbox.Spec.State
    
    // 记录状态转换
    h.Recorder.Eventf(devbox, corev1.EventTypeNormal,
        "StateChanging",
        "Devbox state changing from %s to %s",
        currentState, targetState)
    
    // 执行状态转换
    // ...
    
    h.Recorder.Eventf(devbox, corev1.EventTypeNormal,
        "StateChanged",
        "Devbox state changed to %s successfully",
        targetState)
    
    return nil
}
```

---

## 3. 最佳实践

### 3.1 ✅ 使用有意义的 Reason

```go
// ✅ 正确：清晰的 Reason
r.recorder.Event(devbox, corev1.EventTypeNormal, "PodCreated", "Pod created successfully")
r.recorder.Event(devbox, corev1.EventTypeWarning, "CreatePodFailed", "Failed to create pod")
r.recorder.Event(devbox, corev1.EventTypeNormal, "PhaseChanged", "Phase changed to Running")

// ❌ 错误：模糊的 Reason
r.recorder.Event(devbox, corev1.EventTypeNormal, "Success", "OK")
r.recorder.Event(devbox, corev1.EventTypeWarning, "Error", "Failed")
```

### 3.2 ✅ 提供详细的 Message

```go
// ✅ 正确：详细的错误信息
r.recorder.Eventf(devbox, corev1.EventTypeWarning,
    "CreatePodFailed",
    "Failed to create pod %s/%s: %v",
    devbox.Namespace, devbox.Name, err)

// ❌ 错误：信息不足
r.recorder.Event(devbox, corev1.EventTypeWarning, "Failed", "Error")
```

### 3.3 ✅ 避免频繁记录相同事件

```go
// ✅ 正确：只在状态变化时记录
func (r *DevboxReconciler) syncDevboxPhase(ctx context.Context, devbox *Devbox) error {
    oldPhase := devbox.Status.Phase
    newPhase := DerivePhase(...)
    
    if oldPhase != newPhase {
        // 只在 Phase 变化时记录
        r.Recorder.Eventf(devbox, corev1.EventTypeNormal,
            "PhaseChanged",
            "Phase changed from %s to %s", oldPhase, newPhase)
    }
    
    // 更新 Status
    // ...
}

// ❌ 错误：每次 Reconcile 都记录
func (r *DevboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 每次都记录，导致 Event 泛滥
    r.recorder.Event(devbox, corev1.EventTypeNormal, "Reconciling", "Reconciling devbox")
    // ...
}
```

### 3.4 ✅ 使用不同的 EventType

```go
// ✅ 正确：根据严重程度选择类型
// Normal: 正常操作
r.recorder.Event(obj, corev1.EventTypeNormal, "Created", "Resource created")
r.recorder.Event(obj, corev1.EventTypeNormal, "Updated", "Resource updated")
r.recorder.Event(obj, corev1.EventTypeNormal, "Synced", "Resource synced")

// Warning: 错误、失败、异常
r.recorder.Event(obj, corev1.EventTypeWarning, "CreateFailed", "Failed to create")
r.recorder.Event(obj, corev1.EventTypeWarning, "UpdateFailed", "Failed to update")
r.recorder.Event(obj, corev1.EventTypeWarning, "ValidationFailed", "Validation failed")
```

---

## 4. 查看 Event

### 4.1 kubectl 命令

```bash
# 查看特定资源的 Event
kubectl describe devbox my-devbox

# 查看所有 Event
kubectl get events

# 查看特定 Namespace 的 Event
kubectl get events -n user-ns

# 按时间排序
kubectl get events --sort-by='.lastTimestamp'

# 只看 Warning
kubectl get events --field-selector type=Warning
```

### 4.2 Event 示例

```yaml
LAST SEEN   TYPE      REASON           OBJECT              MESSAGE
2m          Normal    PodCreated       devbox/my-devbox    Pod created successfully
1m          Warning   CreatePodFailed  devbox/my-devbox    Failed to create pod: insufficient resources
30s         Normal    PhaseChanged     devbox/my-devbox    Phase changed from Pending to Running
```

---

## 5. 总结

### 核心要点

1. ✅ **初始化 Recorder** - 在 SetupWithManager 中
2. ✅ **使用清晰的 Reason** - 便于过滤和查询
3. ✅ **提供详细的 Message** - 包含上下文信息
4. ✅ **避免事件泛滥** - 只记录重要操作
5. ✅ **区分 Normal 和 Warning** - 根据严重程度

### 适用场景

| 场景 | 是否记录 Event |
|------|--------------|
| 资源创建成功 | ✅ 是 |
| 资源更新失败 | ✅ 是 |
| Phase 变化 | ✅ 是 |
| 每次 Reconcile | ❌ 否 |
| 调试日志 | ❌ 否（使用 Logger） |

### 与其他模式的配合

- **Request-Driven**: 记录操作请求的执行结果
- **Finalizer**: 记录清理操作
- **Pipeline**: 在每个步骤记录关键事件

### 参考资源

- [Event API 文档](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/)
- [Sealos Terminal Controller](https://github.com/labring/sealos/blob/main/controllers/terminal/controllers/terminal_controller.go)
