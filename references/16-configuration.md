# Configuration Loading 模式

**复杂度**: 🟢 简单  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 环境配置、动态配置、多环境部署

## 概述

Configuration Loading 模式通过环境变量加载控制器配置，支持默认值和类型转换，使控制器可以在不同环境中灵活部署。

## 快速开始

```go
// 从环境变量加载配置
func (r *ObjectStorageBucketReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.Logger = ctrl.Log.WithName("objectstoragebucket-controller")
    
    // 读取配置（带默认值）
    detectionCycleSeconds := env.GetInt64EnvWithDefault("OSBDetectionCycleSeconds", 300)
    r.OSBDetectionCycle = time.Duration(detectionCycleSeconds) * time.Second
    
    r.InternalEndpoint = env.GetEnvWithDefault("INTERNAL_ENDPOINT", "")
    r.ExternalEndpoint = env.GetEnvWithDefault("EXTERNAL_ENDPOINT", "")
    r.OSNamespace = env.GetEnvWithDefault("NAMESPACE", "objectstorage-system")
    
    // 验证必需配置
    if r.InternalEndpoint == "" {
        return fmt.Errorf("INTERNAL_ENDPOINT is required")
    }
    
    r.Logger.Info("controller configured",
        "detectionCycle", r.OSBDetectionCycle,
        "internalEndpoint", r.InternalEndpoint,
        "namespace", r.OSNamespace)
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&objectstoragev1.ObjectStorageBucket{}).
        Complete(r)
}
```

---

## 1. Helper 函数

### 1.1 基本类型

```go
// controllers/pkg/utils/env/env.go
package env

import (
    "os"
    "strconv"
    "time"
)

// GetEnvWithDefault 获取字符串环境变量
func GetEnvWithDefault(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

// GetIntEnvWithDefault 获取整数环境变量
func GetIntEnvWithDefault(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if parsed, err := strconv.Atoi(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}

// GetInt64EnvWithDefault 获取 int64 环境变量
func GetInt64EnvWithDefault(key string, defaultValue int64) int64 {
    if value := os.Getenv(key); value != "" {
        if parsed, err := strconv.ParseInt(value, 10, 64); err == nil {
            return parsed
        }
    }
    return defaultValue
}

// GetBoolEnvWithDefault 获取布尔环境变量
func GetBoolEnvWithDefault(key string, defaultValue bool) bool {
    if value := os.Getenv(key); value != "" {
        if parsed, err := strconv.ParseBool(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}

// GetDurationEnvWithDefault 获取时间间隔环境变量
func GetDurationEnvWithDefault(key string, defaultValue time.Duration) time.Duration {
    if value := os.Getenv(key); value != "" {
        if parsed, err := time.ParseDuration(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}
```

---

## 2. 使用方法

### 2.1 在 SetupWithManager 中加载

```go
type DevboxReconciler struct {
    client.Client
    Scheme               *runtime.Scheme
    Logger               logr.Logger
    Recorder             record.EventRecorder
    NodeName             string
    CommitImageRegistry  string
    DefaultBaseImage     string
    MaxCPULimitRatio     float64
    MaxMemoryLimitRatio  float64
}

func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.Logger = ctrl.Log.WithName("devbox-controller")
    r.Recorder = mgr.GetEventRecorderFor("devbox-controller")
    
    // 从环境变量加载配置
    r.NodeName = env.GetEnvWithDefault("NODE_NAME", "")
    r.CommitImageRegistry = env.GetEnvWithDefault("COMMIT_IMAGE_REGISTRY", "docker.io")
    r.DefaultBaseImage = env.GetEnvWithDefault("DEFAULT_BASE_IMAGE", "ubuntu:22.04")
    
    maxCPULimitRatio := env.GetFloat64EnvWithDefault("MAX_CPU_LIMIT_RATIO", 0.8)
    r.MaxCPULimitRatio = maxCPULimitRatio
    
    maxMemoryLimitRatio := env.GetFloat64EnvWithDefault("MAX_MEMORY_LIMIT_RATIO", 0.8)
    r.MaxMemoryLimitRatio = maxMemoryLimitRatio
    
    // 验证配置
    if r.NodeName == "" {
        return fmt.Errorf("NODE_NAME is required")
    }
    
    // 打印配置
    r.Logger.Info("devbox controller configured",
        "nodeName", r.NodeName,
        "commitImageRegistry", r.CommitImageRegistry,
        "maxCPULimitRatio", r.MaxCPULimitRatio,
        "maxMemoryLimitRatio", r.MaxMemoryLimitRatio)
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}
```

### 2.2 配置结构体

```go
// 使用结构体组织配置
type Config struct {
    NodeName             string
    CommitImageRegistry  string
    DefaultBaseImage     string
    MaxCPULimitRatio     float64
    MaxMemoryLimitRatio  float64
    DetectionCycle       time.Duration
}

func LoadConfig() (*Config, error) {
    cfg := &Config{
        NodeName:             env.GetEnvWithDefault("NODE_NAME", ""),
        CommitImageRegistry:  env.GetEnvWithDefault("COMMIT_IMAGE_REGISTRY", "docker.io"),
        DefaultBaseImage:     env.GetEnvWithDefault("DEFAULT_BASE_IMAGE", "ubuntu:22.04"),
        MaxCPULimitRatio:     env.GetFloat64EnvWithDefault("MAX_CPU_LIMIT_RATIO", 0.8),
        MaxMemoryLimitRatio:  env.GetFloat64EnvWithDefault("MAX_MEMORY_LIMIT_RATIO", 0.8),
        DetectionCycle:       env.GetDurationEnvWithDefault("DETECTION_CYCLE", 5*time.Minute),
    }
    
    // 验证
    if cfg.NodeName == "" {
        return nil, fmt.Errorf("NODE_NAME is required")
    }
    
    return cfg, nil
}

// 在 SetupWithManager 中使用
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    cfg, err := LoadConfig()
    if err != nil {
        return err
    }
    
    r.Config = cfg
    r.Logger.Info("controller configured", "config", cfg)
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}
```

---

## 3. 实际案例

### 案例 1：ObjectStorage 控制器配置

```go
// controllers/objectstorage/controllers/objectstoragebucket_controller.go
const (
    OSBDetectionCycleEnv = "OSBDetectionCycleSeconds"
    InternalEndpointEnv  = "INTERNAL_ENDPOINT"
    ExternalEndpointEnv  = "EXTERNAL_ENDPOINT"
    NamespaceEnv         = "NAMESPACE"
    AdminSecretEnv       = "ADMIN_SECRET"
)

type ObjectStorageBucketReconciler struct {
    client.Client
    Scheme            *runtime.Scheme
    Logger            logr.Logger
    OSClient          *minio.Client
    OSAdminClient     *madmin.AdminClient
    OSNamespace       string
    OSAdminSecret     string
    OSBDetectionCycle time.Duration
    InternalEndpoint  string
    ExternalEndpoint  string
}

func (r *ObjectStorageBucketReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.Logger = ctrl.Log.WithName("objectstoragebucket-controller")
    
    // 加载配置
    detectionCycleSeconds := env.GetInt64EnvWithDefault(OSBDetectionCycleEnv, 300)
    r.OSBDetectionCycle = time.Duration(detectionCycleSeconds) * time.Second
    
    r.InternalEndpoint = env.GetEnvWithDefault(InternalEndpointEnv, "")
    r.ExternalEndpoint = env.GetEnvWithDefault(ExternalEndpointEnv, "")
    r.OSNamespace = env.GetEnvWithDefault(NamespaceEnv, "objectstorage-system")
    r.OSAdminSecret = env.GetEnvWithDefault(AdminSecretEnv, "objectstorage-admin")
    
    // 验证
    if r.InternalEndpoint == "" {
        return fmt.Errorf("%s is required", InternalEndpointEnv)
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&objectstoragev1.ObjectStorageBucket{}).
        Complete(r)
}
```

### 案例 2：Account 控制器配置

```go
// controllers/account/main.go
const (
    EnvAccountNamespace     = "ACCOUNT_NAMESPACE"
    EnvSubscriptionEnabled  = "SUBSCRIPTION_ENABLED"
    EnvJwtSecret            = "ACCOUNT_API_JWT_SECRET"
    EnvDatabaseURI          = "DATABASE_URI"
    EnvConcurrent           = "CONCURRENT_RECONCILES"
)

func main() {
    // 加载配置
    accountNamespace := env.GetEnvWithDefault(EnvAccountNamespace, "sealos-system")
    subscriptionEnabled := env.GetBoolEnvWithDefault(EnvSubscriptionEnabled, false)
    jwtSecret := env.GetEnvWithDefault(EnvJwtSecret, "")
    databaseURI := env.GetEnvWithDefault(EnvDatabaseURI, "")
    concurrent := env.GetIntEnvWithDefault(EnvConcurrent, 10)
    
    // 验证
    if databaseURI == "" {
        setupLog.Error(errors.New("DATABASE_URI is required"), "")
        os.Exit(1)
    }
    
    setupLog.Info("account controller config",
        "namespace", accountNamespace,
        "subscriptionEnabled", subscriptionEnabled,
        "concurrent", concurrent)
    
    // 创建控制器
    // ...
}
```

---

## 4. 最佳实践

### 4.1 ✅ 使用常量定义环境变量名

```go
// ✅ 正确：使用常量
const (
    EnvNodeName = "NODE_NAME"
    EnvRegistry = "COMMIT_IMAGE_REGISTRY"
)

nodeName := env.GetEnvWithDefault(EnvNodeName, "")

// ❌ 错误：使用字符串字面量
nodeName := env.GetEnvWithDefault("NODE_NAME", "")  // 容易拼写错误
```

### 4.2 ✅ 提供合理的默认值

```go
// ✅ 正确：合理的默认值
detectionCycle := env.GetDurationEnvWithDefault("DETECTION_CYCLE", 5*time.Minute)
maxRetries := env.GetIntEnvWithDefault("MAX_RETRIES", 3)
enableFeature := env.GetBoolEnvWithDefault("ENABLE_FEATURE", false)

// ❌ 错误：没有默认值或默认值不合理
detectionCycle := env.GetDurationEnvWithDefault("DETECTION_CYCLE", 0)  // 0 不合理
```

### 4.3 ✅ 验证必需配置

```go
// ✅ 正确：验证必需配置
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.NodeName = env.GetEnvWithDefault("NODE_NAME", "")
    
    if r.NodeName == "" {
        return fmt.Errorf("NODE_NAME is required")
    }
    
    // ...
}

// ❌ 错误：不验证，运行时才发现问题
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.NodeName = env.GetEnvWithDefault("NODE_NAME", "")
    // 没有验证，Reconcile 时才报错
    // ...
}
```

### 4.4 ✅ 打印配置日志

```go
// ✅ 正确：打印配置，便于调试
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // 加载配置
    r.NodeName = env.GetEnvWithDefault("NODE_NAME", "")
    r.MaxCPULimitRatio = env.GetFloat64EnvWithDefault("MAX_CPU_LIMIT_RATIO", 0.8)
    
    // 打印配置
    r.Logger.Info("devbox controller configured",
        "nodeName", r.NodeName,
        "maxCPULimitRatio", r.MaxCPULimitRatio)
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}
```

---

## 5. 配置类型

### 5.1 字符串配置

```go
// 端点、路径、名称等
endpoint := env.GetEnvWithDefault("ENDPOINT", "http://localhost:8080")
namespace := env.GetEnvWithDefault("NAMESPACE", "default")
imageName := env.GetEnvWithDefault("IMAGE", "nginx:latest")
```

### 5.2 数值配置

```go
// 并发数、重试次数、限制等
concurrent := env.GetIntEnvWithDefault("CONCURRENT", 10)
maxRetries := env.GetIntEnvWithDefault("MAX_RETRIES", 3)
timeout := env.GetInt64EnvWithDefault("TIMEOUT_SECONDS", 30)
ratio := env.GetFloat64EnvWithDefault("CPU_RATIO", 0.8)
```

### 5.3 时间配置

```go
// 间隔、超时、TTL 等
detectionCycle := env.GetDurationEnvWithDefault("DETECTION_CYCLE", 5*time.Minute)
timeout := env.GetDurationEnvWithDefault("TIMEOUT", 30*time.Second)
ttl := env.GetDurationEnvWithDefault("TTL", 24*time.Hour)

// 支持的格式: "300s", "5m", "1h", "24h"
```

### 5.4 布尔配置

```go
// 功能开关
enableFeature := env.GetBoolEnvWithDefault("ENABLE_FEATURE", false)
debugMode := env.GetBoolEnvWithDefault("DEBUG", false)
dryRun := env.GetBoolEnvWithDefault("DRY_RUN", false)

// 支持的值: "true", "false", "1", "0", "yes", "no"
```

---

## 6. 部署配置

### 6.1 Deployment 环境变量

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devbox-controller
spec:
  template:
    spec:
      containers:
      - name: manager
        image: ghcr.io/labring/sealos-devbox-controller:latest
        env:
        # 必需配置
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        
        # 可选配置（有默认值）
        - name: COMMIT_IMAGE_REGISTRY
          value: "docker.io"
        
        - name: MAX_CPU_LIMIT_RATIO
          value: "0.8"
        
        - name: DETECTION_CYCLE
          value: "5m"
        
        # 从 Secret 读取
        - name: DATABASE_URI
          valueFrom:
            secretKeyRef:
              name: devbox-config
              key: database-uri
```

### 6.2 ConfigMap 配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: devbox-config
data:
  NODE_NAME: "node-1"
  COMMIT_IMAGE_REGISTRY: "ghcr.io"
  MAX_CPU_LIMIT_RATIO: "0.8"
  DETECTION_CYCLE: "5m"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devbox-controller
spec:
  template:
    spec:
      containers:
      - name: manager
        envFrom:
        - configMapRef:
            name: devbox-config
```

---

## 7. 最佳实践

### 7.1 ✅ 配置分层

```go
// ✅ 正确：配置分层
// 1. 硬编码默认值（最低优先级）
const DefaultRegistry = "docker.io"

// 2. 环境变量（中等优先级）
registry := env.GetEnvWithDefault("REGISTRY", DefaultRegistry)

// 3. ConfigMap/Secret（高优先级）
// 4. 命令行参数（最高优先级）
```

### 7.2 ✅ 敏感信息使用 Secret

```go
// ✅ 正确：敏感信息从 Secret 读取
env:
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef:
      name: database-secret
      key: password

// ❌ 错误：敏感信息明文
env:
- name: DATABASE_PASSWORD
  value: "my-password"  # ❌ 不安全
```

### 7.3 ✅ 配置验证

```go
// ✅ 正确：完整的验证
func (r *DevboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // 加载配置
    r.NodeName = env.GetEnvWithDefault("NODE_NAME", "")
    r.MaxCPULimitRatio = env.GetFloat64EnvWithDefault("MAX_CPU_LIMIT_RATIO", 0.8)
    
    // 验证必需配置
    if r.NodeName == "" {
        return fmt.Errorf("NODE_NAME is required")
    }
    
    // 验证范围
    if r.MaxCPULimitRatio <= 0 || r.MaxCPULimitRatio > 1 {
        return fmt.Errorf("MAX_CPU_LIMIT_RATIO must be between 0 and 1, got %f", r.MaxCPULimitRatio)
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&devboxv1alpha2.Devbox{}).
        Complete(r)
}
```

---

## 8. 总结

### 核心要点

1. ✅ **使用环境变量** - 灵活配置
2. ✅ **提供默认值** - 降低配置复杂度
3. ✅ **验证配置** - 启动时发现问题
4. ✅ **打印配置** - 便于调试
5. ✅ **敏感信息用 Secret** - 安全性

### 适用场景

| 配置类型 | 推荐方式 |
|---------|---------|
| 端点地址 | 环境变量 |
| 超时时间 | 环境变量 + 默认值 |
| 功能开关 | 环境变量 + 默认值 |
| 密码/Token | Secret |
| 大量配置 | ConfigMap |

### 与其他模式的配合

- **Scheduled Task**: 配置任务间隔
- **Concurrent Reconcile**: 配置并发数
- **External Service Integration**: 配置外部服务端点

### 参考资源

- [Sealos Devbox Controller](https://github.com/labring/sealos/blob/main/controllers/devbox/internal/controller/devbox_controller.go)
- [Sealos ObjectStorage Controller](https://github.com/labring/sealos/tree/main/controllers/objectstorage/controllers)
- [Kubernetes Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
