# Scheduled Task 模式

**复杂度**: 🟡 中等  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 周期性任务、后台任务、定时计费

## 概述

Scheduled Task 模式通过实现 `manager.Runnable` 接口，在控制器进程中运行后台周期性任务。常用于计费、清理、统计等不需要 Watch 资源变化的场景。

## 快速开始

```go
// 1. 定义 Task Runner
type BillingTaskRunner struct {
    DBClient database.Interface
    Logger   logr.Logger
    *AccountReconciler
}

// 2. 实现 Runnable 接口
func (r *BillingTaskRunner) Start(ctx context.Context) error {
    ticker := time.NewTicker(10 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            r.Logger.Info("start billing", "time", time.Now())
            if err := r.BillingCVM(); err != nil {
                r.Logger.Error(err, "billing failed")
            }
        case <-ctx.Done():
            return nil
        }
    }
}

// 3. 注册到 Manager
func (r *AccountReconciler) SetupWithManager(mgr ctrl.Manager) error {
    if err := mgr.Add(&BillingTaskRunner{
        DBClient:          r.DBClient,
        Logger:            r.Logger,
        AccountReconciler: r,
    }); err != nil {
        return err
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&accountv1.Account{}).
        Complete(r)
}
```

---

## 实现方式

### 方式一：使用 Ticker

```go
// controllers/account/main.go
type CVMTaskRunner struct {
    DBClient database.Interface
    Logger   logr.Logger
    *AccountReconciler
}

func (r *CVMTaskRunner) Start(ctx context.Context) error {
    // 从环境变量读取间隔
    interval := env.GetDurationEnvWithDefault("BILLING_CVM_INTERVAL", 10*time.Minute)
    ticker := time.NewTicker(interval)
    defer func() {
        ticker.Stop()
        r.Logger.Info("stop billing cvm")
    }()
    
    for {
        select {
        case <-ticker.C:
            r.Logger.Info("start billing cvm", "time", time.Now().Format(time.RFC3339))
            err := r.BillingCVM()
            if err != nil {
                r.Logger.Error(err, "fail to billing cvm")
            }
            r.Logger.Info("end billing cvm", "time", time.Now().Format(time.RFC3339))
        case <-ctx.Done():
            return nil
        }
    }
}

// 业务逻辑
func (r *AccountReconciler) BillingCVM() error {
    // 查询数据库
    cvms, err := r.DBClient.GetAllCVMs()
    if err != nil {
        return err
    }
    
    // 计费逻辑
    for _, cvm := range cvms {
        if err := r.billCVM(cvm); err != nil {
            r.Logger.Error(err, "failed to bill cvm", "cvm", cvm.ID)
        }
    }
    
    return nil
}
```

### 方式二：使用 Cron

```go
import "github.com/robfig/cron/v3"

type CronTaskRunner struct {
    Logger logr.Logger
    cron   *cron.Cron
}

func (r *CronTaskRunner) Start(ctx context.Context) error {
    r.cron = cron.New()
    
    // 每天凌晨 2 点执行
    r.cron.AddFunc("0 2 * * *", func() {
        r.Logger.Info("start daily cleanup")
        if err := r.dailyCleanup(); err != nil {
            r.Logger.Error(err, "daily cleanup failed")
        }
    })
    
    // 每小时执行
    r.cron.AddFunc("0 * * * *", func() {
        r.Logger.Info("start hourly billing")
        if err := r.hourlyBilling(); err != nil {
            r.Logger.Error(err, "hourly billing failed")
        }
    })
    
    r.cron.Start()
    
    <-ctx.Done()
    r.cron.Stop()
    return nil
}
```

---

## 实际案例

### 案例 1：Account 控制器的 CVM 计费

```go
// controllers/account/main.go
func main() {
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }
    
    // 创建 Account Reconciler
    accountReconciler := &controllers.AccountReconciler{
        Client:   mgr.GetClient(),
        Scheme:   mgr.GetScheme(),
        DBClient: dbClient,
    }
    
    if err = accountReconciler.SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "Account")
        os.Exit(1)
    }
    
    // 注册 CVM 计费任务
    if env.GetEnvWithDefault("ENABLE_CVM_BILLING", "false") == "true" {
        cvmTaskRunner := &controllers.CVMTaskRunner{
            DBClient:          dbClient,
            Logger:            ctrl.Log.WithName("CVMTaskRunner"),
            AccountReconciler: accountReconciler,
        }
        if err := mgr.Add(cvmTaskRunner); err != nil {
            setupLog.Error(err, "unable to add cvm task runner")
            os.Exit(1)
        }
    }
    
    // 启动 Manager
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

### 案例 2：Billing 任务

```go
// controllers/account/controllers/billing_controller.go
type BillingTaskRunner struct {
    BillingReconciler *BillingReconciler
}

func (r *BillingTaskRunner) Start(ctx context.Context) error {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            r.BillingReconciler.Logger.Info("start billing")
            
            // 获取所有需要计费的 Account
            accountList := &accountv1.AccountList{}
            if err := r.BillingReconciler.List(ctx, accountList); err != nil {
                r.BillingReconciler.Logger.Error(err, "failed to list accounts")
                continue
            }
            
            // 并发计费
            var wg sync.WaitGroup
            for i := range accountList.Items {
                wg.Add(1)
                go func(account *accountv1.Account) {
                    defer wg.Done()
                    if err := r.BillingReconciler.billAccount(ctx, account); err != nil {
                        r.BillingReconciler.Logger.Error(err, "failed to bill account", 
                            "account", account.Name)
                    }
                }(&accountList.Items[i])
            }
            wg.Wait()
            
            r.BillingReconciler.Logger.Info("billing completed")
        case <-ctx.Done():
            return nil
        }
    }
}
```

---

## 最佳实践

### 1. ✅ 使用环境变量配置间隔

```go
// ✅ 正确：从环境变量读取
interval := env.GetDurationEnvWithDefault("BILLING_INTERVAL", 10*time.Minute)
ticker := time.NewTicker(interval)

// ❌ 错误：硬编码
ticker := time.NewTicker(10 * time.Minute)
```

### 2. ✅ 优雅关闭

```go
func (r *TaskRunner) Start(ctx context.Context) error {
    ticker := time.NewTicker(interval)
    defer func() {
        ticker.Stop()
        r.Logger.Info("task runner stopped")
    }()
    
    for {
        select {
        case <-ticker.C:
            r.doWork()
        case <-ctx.Done():
            return nil  // 优雅退出
        }
    }
}
```

### 3. ✅ 错误处理

```go
func (r *TaskRunner) Start(ctx context.Context) error {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            // 捕获 panic
            func() {
                defer func() {
                    if r := recover(); r != nil {
                        r.Logger.Error(fmt.Errorf("%v", r), "task panic")
                    }
                }()
                
                if err := r.doWork(); err != nil {
                    r.Logger.Error(err, "task failed")
                    // 继续执行，不退出
                }
            }()
        case <-ctx.Done():
            return nil
        }
    }
}
```

### 4. ✅ 添加监控指标

```go
import "github.com/prometheus/client_golang/prometheus"

var (
    taskDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "task_duration_seconds",
            Help: "Task execution duration",
        },
        []string{"task_name"},
    )
    taskErrors = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "task_errors_total",
            Help: "Total number of task errors",
        },
        []string{"task_name"},
    )
)

func (r *TaskRunner) Start(ctx context.Context) error {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            start := time.Now()
            err := r.doWork()
            duration := time.Since(start).Seconds()
            
            taskDuration.WithLabelValues("billing").Observe(duration)
            if err != nil {
                taskErrors.WithLabelValues("billing").Inc()
                r.Logger.Error(err, "task failed")
            }
        case <-ctx.Done():
            return nil
        }
    }
}
```

---

## 总结

### 核心要点

1. ✅ **实现 Runnable 接口** - `Start(context.Context) error`
2. ✅ **监听 ctx.Done()** - 优雅关闭
3. ✅ **使用环境变量配置** - 灵活调整间隔
4. ✅ **错误处理** - 捕获 panic，记录日志
5. ✅ **添加监控** - 记录执行时间和错误

### 适用场景

| 场景 | 是否使用 Scheduled Task |
|------|----------------------|
| 周期性计费 | ✅ 是 |
| 定时清理 | ✅ 是 |
| 统计报表 | ✅ 是 |
| 资源同步 | ❌ 否（使用 Controller Watch） |
| 事件驱动 | ❌ 否（使用 Controller） |

### 与其他模式的配合

- **External Service Integration**: 任务中访问外部服务
- **Configuration Loading**: 从环境变量读取配置
- **Event Recording**: 记录任务执行事件

### 参考资源

- [Runnable 接口文档](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Runnable)
- [Sealos Account Controller 实现](https://github.com/labring/sealos/blob/main/controllers/account/main.go)
