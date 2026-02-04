# Sealos Kubebuilder 设计模式文档

本系列文档详细介绍了 Sealos 项目中使用的 Kubebuilder 开发模式和最佳实践。这些模式是从多个生产级控制器（Account、Devbox、User、Terminal、ObjectStorage、License 等）中提炼总结而来。

## 目录

### 核心模式

1. [RetryOnConflict 模式](./01-retry-on-conflict.md) - 乐观锁冲突重试机制
2. [CreateOrUpdate 模式](./02-create-or-update.md) - 声明式资源管理
3. [Finalizer 模式](./03-finalizer.md) - 资源清理与级联删除
4. [Custom Predicate 模式](./04-custom-predicate.md) - 事件过滤与性能优化

### 性能优化模式

5. [Field Indexer 模式](./05-field-indexer.md) - 缓存索引优化查询
6. [Concurrent Reconcile 模式](./06-concurrent-reconcile.md) - 并发控制与限流

### 架构模式

7. [Pipeline 模式](./07-pipeline.md) - 多步骤编排
8. [Request-Driven 模式](./08-request-driven.md) - 请求驱动的异步操作
9. [Phase Derivation 模式](./09-phase-derivation.md) - 状态推导与聚合

### 集成模式

10. [External Service Integration 模式](./10-external-service.md) - 外部服务集成
11. [Scheduled Task 模式](./11-scheduled-task.md) - 后台定时任务
12. [Event Recording 模式](./12-event-recording.md) - 事件记录与审计

### 资源管理模式

13. [Owner Reference 模式](./13-owner-reference.md) - 资源关系与级联删除
14. [TTL/Expiration 模式](./14-ttl-expiration.md) - 资源过期与自动清理
15. [Resource Deletion 模式](./15-resource-deletion.md) - 安全删除与 UID 检查

### 工程实践

16. [Configuration Loading 模式](./16-configuration.md) - 环境配置管理
17. [Recommended Labels 模式](./17-labels.md) - 标准化标签规范
18. [Error Handling 模式](./18-error-handling.md) - 错误处理最佳实践

## 快速导航

### 按场景查找

| 场景 | 推荐模式 |
|------|---------|
| 更新资源状态 | [RetryOnConflict](./01-retry-on-conflict.md) |
| 创建子资源 | [CreateOrUpdate](./02-create-or-update.md) + [Owner Reference](./13-owner-reference.md) |
| 资源删除前清理 | [Finalizer](./03-finalizer.md) |
| 减少不必要的 Reconcile | [Custom Predicate](./04-custom-predicate.md) |
| 优化 List 查询 | [Field Indexer](./05-field-indexer.md) |
| 复杂多步骤操作 | [Pipeline](./07-pipeline.md) |
| 一次性异步操作 | [Request-Driven](./08-request-driven.md) |
| 与外部系统交互 | [External Service Integration](./10-external-service.md) |
| 周期性任务 | [Scheduled Task](./11-scheduled-task.md) |
| 资源自动过期 | [TTL/Expiration](./14-ttl-expiration.md) |

### 按控制器查找

| 控制器 | 主要使用的模式 |
|--------|---------------|
| **Devbox** | RetryOnConflict, Custom Predicate, Field Indexer, Phase Derivation, Resource Deletion |
| **User** | Pipeline, Finalizer, Request-Driven, Concurrent Reconcile |
| **Account** | Scheduled Task, External Service Integration, Field Indexer |
| **Terminal** | CreateOrUpdate, Owner Reference, TTL/Expiration, Event Recording |
| **ObjectStorage** | External Service Integration, Configuration Loading |
| **License** | Finalizer, Custom Predicate, Configuration Loading |

## 文档约定

### 代码示例

- ✅ 标记表示推荐的做法
- ❌ 标记表示应避免的做法
- ⚠️ 标记表示需要注意的情况

### 复杂度标记

- 🟢 **简单** - 可直接使用，无需修改
- 🟡 **中等** - 需要根据场景调整
- 🔴 **复杂** - 需要深入理解原理

### 生产就绪度

- 🚀 **生产验证** - 已在 Sealos 生产环境验证
- 🧪 **实验性** - 新引入的模式，需要更多验证

## 贡献指南

如果你发现新的模式或对现有文档有改进建议，欢迎提交 PR。请确保：

1. 提供完整的代码示例
2. 说明适用场景和限制
3. 包含实际的使用案例
4. 遵循现有文档的格式

## 参考资源

- [Kubebuilder 官方文档](https://book.kubebuilder.io/)
- [Controller Runtime 文档](https://pkg.go.dev/sigs.k8s.io/controller-runtime)
- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [Sealos Kubebuilder Skill](./../.claude/skills/sealos-kubebuilder/SKILL.md)

## 版本历史

- **v1.0** (2026-01) - 初始版本，包含 18 种核心模式
