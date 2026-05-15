# bugs/ — 问题分类记录

按"根因类型"归档，不按时间排序。每个分类下一个 markdown，记录：
- 问题现象
- 根因
- 修复 commit
- **如何避免**（重点 — 决定下次是不是同一个坑）

## 为什么这样组织

按时间记 daily log 容易让"同一类问题反复出现"被淹没。按根因分类后，下次遇到
"接口报 500"先翻 [`schema-drift.md`](schema-drift.md)，遇到"加完看不到"先翻
[`frontend-backend-contract.md`](frontend-backend-contract.md) — 几分钟内能找到同类先例。

## 当前分类

| 文件 | 范围 | 案例数 |
|---|---|---|
| [`frontend-backend-contract.md`](frontend-backend-contract.md) | 前后端字段名 / 类型 / 路径不对齐；接口契约漂移 | 5 |
| [`infra-and-network.md`](infra-and-network.md) | 容器网络 / DNS / Nginx upstream / 端口冲突 | 2 |
| [`db-replication-and-routing.md`](db-replication-and-routing.md) | ProxySQL 路由 / 主从延迟 / read-after-write | 4 |
| [`schema-drift.md`](schema-drift.md) | DDL 加列后 SELECT 没跟上；存储格式与解析器不匹配 | 3 |
| [`placeholder-shipped.md`](placeholder-shipped.md) | 占位代码混进了关键路径，被当成"做完了" | 3 |

## 写作约定

```markdown
## #N 标题（一句话症状）

**症状**：用户/CI 视角看到的现象（带 HTTP code、错误消息）
**触发**：必现 / 偶现（条件）
**根因**：底层是什么坏了（1-2 段，引用具体代码或配置）
**修复**：commit hash + 具体改动概括
**如何避免**：以后类似问题怎么提前发现 / 怎么写代码不踩
**相关**：（可选）关联 ADR / 调查文档 / 其他 bug
```

新增条目时同时更新这里 README 的"案例数"列。
