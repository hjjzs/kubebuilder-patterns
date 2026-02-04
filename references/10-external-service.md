# External Service Integration 模式

**复杂度**: 🔴 复杂  
**生产就绪度**: 🚀 生产验证  
**适用场景**: 对象存储、数据库、消息队列等外部服务集成

## 概述

External Service Integration 模式用于将 Kubernetes 控制器与外部服务（MinIO、数据库、消息队列等）集成，实现 Kubernetes 资源与外部资源的同步管理。

## 快速开始

```go
// ObjectStorage 控制器集成 MinIO
type ObjectStorageBucketReconciler struct {
    client.Client
    Scheme            *runtime.Scheme
    Logger            logr.Logger
    OSClient          *minio.Client        // MinIO 客户端
    OSAdminClient     *madmin.AdminClient  // MinIO Admin 客户端
    InternalEndpoint  string
}

func (r *ObjectStorageBucketReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    bucket := &objectstoragev1.ObjectStorageBucket{}
    if err := r.Get(ctx, req.NamespacedName, bucket); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 1. 初始化外部服务客户端（懒加载）
    if r.OSClient == nil {
        if err := r.initOSClient(ctx); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 2. 同步到外部服务
    bucketName := buildBucketName(bucket.Name, bucket.Namespace)
    
    exists, err := r.OSClient.BucketExists(ctx, bucketName)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    if !exists {
        if err := r.OSClient.MakeBucket(ctx, bucketName, options); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 3. 设置 Bucket 策略
    if err := r.OSClient.SetBucketPolicy(ctx, bucketName, policy); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

---

## 1. 客户端管理

### 1.1 懒加载模式

```go
// controllers/objectstorage/controllers/objectstoragebucket_controller.go
func (r *ObjectStorageBucketReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 懒加载：第一次使用时初始化
    if r.OSClient == nil || r.OSAdminClient == nil {
        secret := &corev1.Secret{}
        if err := r.Get(ctx, client.ObjectKey{
            Name:      r.OSAdminSecret,
            Namespace: r.OSNamespace,
        }, secret); err != nil {
            return ctrl.Result{}, err
        }
        
        endpoint := r.InternalEndpoint
        accessKey := string(secret.Data[AccessKey])
        secretKey := string(secret.Data[SecretKey])
        
        var err error
        if r.OSAdminClient, err = NewOSAdminClient(endpoint, accessKey, secretKey); err != nil {
            r.Logger.Error(err, "failed to create admin client")
            return ctrl.Result{}, err
        }
        
        if r.OSClient, err = NewOSClient(endpoint, accessKey, secretKey); err != nil {
            r.Logger.Error(err, "failed to create client")
            return ctrl.Result{}, err
        }
    }
    
    // 使用客户端
    // ...
}
```

### 1.2 提前初始化模式

```go
// controllers/account/controllers/account_controller.go
type AccountReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Logger   logr.Logger
    DBClient database.Interface  // 数据库客户端
}

func (r *AccountReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.Logger = ctrl.Log.WithName("account-controller")
    
    // 在 SetupWithManager 中初始化数据库连接
    dbURI := env.GetEnvWithDefault("DATABASE_URI", "")
    if dbURI == "" {
        return fmt.Errorf("DATABASE_URI is required")
    }
    
    var err error
    r.DBClient, err = database.NewClient(dbURI)
    if err != nil {
        return fmt.Errorf("failed to connect to database: %w", err)
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&accountv1.Account{}).
        Complete(r)
}
```

---

## 2. 资源同步

### 2.1 创建外部资源

```go
func (r *ObjectStorageBucketReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    bucket := &objectstoragev1.ObjectStorageBucket{}
    if err := r.Get(ctx, req.NamespacedName, bucket); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    bucketName := buildBucketName(bucket.Name, bucket.Namespace)
    
    // 检查 Bucket 是否存在
    exists, err := r.OSClient.BucketExists(ctx, bucketName)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    if !exists {
        // 创建 Bucket
        if err := r.OSClient.MakeBucket(ctx, bucketName, minio.MakeBucketOptions{
            Region: DefaultRegion,
        }); err != nil {
            r.Logger.Error(err, "failed to create bucket", "bucket", bucketName)
            return ctrl.Result{}, err
        }
        r.Logger.Info("bucket created", "bucket", bucketName)
    }
    
    // 设置 Bucket 策略
    policy := buildPolicy(bucket.Spec.Policy, bucketName)
    if err := r.OSClient.SetBucketPolicy(ctx, bucketName, policy); err != nil {
        return ctrl.Result{}, err
    }
    
    // 创建 Service Account
    serviceAccountName := buildSAName(bucketName)
    if err := r.createServiceAccount(ctx, serviceAccountName, bucketName); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

### 2.2 删除外部资源（Finalizer）

```go
func (r *ObjectStorageBucketReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    bucket := &objectstoragev1.ObjectStorageBucket{}
    if err := r.Get(ctx, req.NamespacedName, bucket); err != nil {
        if !errors.IsNotFound(err) {
            return ctrl.Result{}, err
        }
        
        // Bucket CR 已删除，清理外部资源
        bucketName := buildBucketName(req.Name, req.Namespace)
        
        // 1. 清空 Bucket
        objects := r.OSClient.ListObjects(ctx, bucketName, minio.ListObjectsOptions{
            Recursive: true,
        })
        for object := range objects {
            if err := r.OSClient.RemoveObject(ctx, bucketName, object.Key, minio.RemoveObjectOptions{}); err != nil {
                r.Logger.Error(err, "failed to remove object", "object", object.Key)
                return ctrl.Result{}, err
            }
        }
        
        // 2. 删除 Bucket
        if err := r.OSClient.RemoveBucket(ctx, bucketName); err != nil {
            r.Logger.Error(err, "failed to remove bucket", "bucket", bucketName)
            return ctrl.Result{}, err
        }
        
        // 3. 删除 Service Account
        serviceAccountName := buildSAName(bucketName)
        if err := r.OSAdminClient.DeleteServiceAccount(ctx, serviceAccountName); err != nil {
            r.Logger.Error(err, "failed to delete service account")
            return ctrl.Result{}, err
        }
        
        return ctrl.Result{}, nil
    }
    
    // 正常处理
    // ...
}
```

---

## 3. 错误处理

### 3.1 临时错误 vs 永久错误

```go
func (r *ObjectStorageBucketReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    bucket := &objectstoragev1.ObjectStorageBucket{}
    if err := r.Get(ctx, req.NamespacedName, bucket); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 创建 Bucket
    if err := r.OSClient.MakeBucket(ctx, bucketName, options); err != nil {
        // 判断错误类型
        if isTemporaryError(err) {
            // 临时错误：网络问题、服务暂时不可用
            r.Logger.Info("temporary error, will retry", "error", err)
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
        
        if isBucketAlreadyExists(err) {
            // Bucket 已存在，继续处理
            r.Logger.Info("bucket already exists", "bucket", bucketName)
        } else {
            // 永久错误：权限不足、配额超限等
            r.Logger.Error(err, "permanent error")
            r.Recorder.Event(bucket, corev1.EventTypeWarning, "CreateFailed", err.Error())
            return ctrl.Result{}, err
        }
    }
    
    return ctrl.Result{}, nil
}

func isTemporaryError(err error) bool {
    // MinIO 错误码判断
    minioErr, ok := err.(minio.ErrorResponse)
    if !ok {
        return false
    }
    
    // 5xx 错误为临时错误
    return minioErr.StatusCode >= 500
}
```

### 3.2 重试策略

```go
import "github.com/labring/sealos/controllers/pkg/utils/retry"

func (r *ObjectStorageBucketReconciler) createBucketWithRetry(
    ctx context.Context,
    bucketName string,
) error {
    return retry.Retry(3, 5*time.Second, func() error {
        return r.OSClient.MakeBucket(ctx, bucketName, options)
    })
}

// retry.Retry 实现
func Retry(tryTimes int, trySleepTime time.Duration, action func() error) error {
    var err error
    for i := 0; i < tryTimes; i++ {
        if err = action(); err == nil {
            return nil
        }
        time.Sleep(trySleepTime * time.Duration(2*i+1))  // 指数退避
    }
    return fmt.Errorf("retry action timeout: %w", err)
}
```

---

## 4. 配置管理

### 4.1 从环境变量读取

```go
// controllers/objectstorage/controllers/objectstoragebucket_controller.go
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
    
    // 从环境变量读取配置
    detectionCycleSeconds := env.GetInt64EnvWithDefault("OSBDetectionCycleSeconds", 300)
    r.OSBDetectionCycle = time.Duration(detectionCycleSeconds) * time.Second
    
    r.InternalEndpoint = env.GetEnvWithDefault("INTERNAL_ENDPOINT", "")
    r.ExternalEndpoint = env.GetEnvWithDefault("EXTERNAL_ENDPOINT", "")
    r.OSNamespace = env.GetEnvWithDefault("NAMESPACE", "objectstorage-system")
    r.OSAdminSecret = env.GetEnvWithDefault("ADMIN_SECRET", "objectstorage-admin")
    
    // 验证必需配置
    if r.InternalEndpoint == "" {
        return fmt.Errorf("INTERNAL_ENDPOINT is required")
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&objectstoragev1.ObjectStorageBucket{}).
        Complete(r)
}
```

### 4.2 从 Secret 读取凭证

```go
func (r *ObjectStorageBucketReconciler) initOSClient(ctx context.Context) error {
    secret := &corev1.Secret{}
    if err := r.Get(ctx, client.ObjectKey{
        Name:      r.OSAdminSecret,
        Namespace: r.OSNamespace,
    }, secret); err != nil {
        return fmt.Errorf("failed to get secret: %w", err)
    }
    
    accessKey := string(secret.Data["accessKey"])
    secretKey := string(secret.Data["secretKey"])
    
    if accessKey == "" || secretKey == "" {
        return fmt.Errorf("accessKey or secretKey is empty")
    }
    
    var err error
    r.OSClient, err = minio.New(r.InternalEndpoint, &minio.Options{
        Creds:  credentials.NewStaticV4(accessKey, secretKey, ""),
        Secure: false,
    })
    if err != nil {
        return fmt.Errorf("failed to create minio client: %w", err)
    }
    
    r.OSAdminClient, err = madmin.New(r.InternalEndpoint, accessKey, secretKey, false)
    if err != nil {
        return fmt.Errorf("failed to create minio admin client: %w", err)
    }
    
    return nil
}
```

---

## 5. 实际案例

### 案例 1：ObjectStorage 控制器

完整的 MinIO 集成示例，参见：
- `controllers/objectstorage/controllers/objectstoragebucket_controller.go`
- `controllers/objectstorage/controllers/objectstorageuser_controller.go`

### 案例 2：Account 控制器

数据库集成示例：
- `controllers/account/controllers/account_controller.go`
- `controllers/pkg/database/` - 数据库接口和实现

---

## 6. 最佳实践

### 6.1 ✅ 使用接口抽象

```go
// ✅ 正确：定义接口，便于测试和替换
type ObjectStorageClient interface {
    BucketExists(ctx context.Context, bucketName string) (bool, error)
    MakeBucket(ctx context.Context, bucketName string, opts minio.MakeBucketOptions) error
    RemoveBucket(ctx context.Context, bucketName string) error
    SetBucketPolicy(ctx context.Context, bucketName string, policy string) error
}

type ObjectStorageBucketReconciler struct {
    client.Client
    OSClient ObjectStorageClient  // 使用接口
}
```

### 6.2 ✅ 连接池管理

```go
// ✅ 正确：复用连接
type AccountReconciler struct {
    client.Client
    DBClient database.Interface  // 连接池
}

// 在 SetupWithManager 中初始化一次
func (r *AccountReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.DBClient, err = database.NewClient(dbURI)  // 创建连接池
    // ...
}
```

### 6.3 ✅ 健康检查

```go
func (r *ObjectStorageBucketReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // 添加健康检查
    if err := mgr.AddHealthzCheck("objectstorage", func(req *http.Request) error {
        if r.OSClient == nil {
            return fmt.Errorf("objectstorage client not initialized")
        }
        
        // 测试连接
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        
        _, err := r.OSClient.ListBuckets(ctx)
        return err
    }); err != nil {
        return err
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&objectstoragev1.ObjectStorageBucket{}).
        Complete(r)
}
```

---

## 7. 总结

### 核心要点

1. ✅ **客户端管理** - 懒加载或提前初始化
2. ✅ **错误处理** - 区分临时和永久错误
3. ✅ **重试策略** - 指数退避
4. ✅ **配置管理** - 从环境变量和 Secret 读取
5. ✅ **资源清理** - 使用 Finalizer

### 适用场景

| 外部服务 | 使用示例 |
|---------|---------|
| 对象存储（MinIO/S3） | ObjectStorage 控制器 |
| 数据库（MySQL/PostgreSQL） | Account 控制器 |
| 消息队列（Kafka/RabbitMQ） | 事件处理 |
| 缓存（Redis） | 性能优化 |

### 参考资源

- [Sealos ObjectStorage Controller](https://github.com/labring/sealos/tree/main/controllers/objectstorage/controllers)
- [Sealos Account Controller](https://github.com/labring/sealos/tree/main/controllers/account/controllers)
- [MinIO Go SDK](https://github.com/minio/minio-go)
