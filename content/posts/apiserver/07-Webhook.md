---
title: "K8s API Server 源码笔记（七）：Webhook 准入"
date: 2026-05-23T07:00:00+08:00
draft: false
tags: ["Kubernetes", "源码分析", "API Server"]
categories: ["云原生"]
series: ["K8s API Server 源码笔记"]
summary: "5 层匹配过滤、Mutating 串行调度、Validating 并行调度、FailurePolicy"
weight: 7
---

## 整体架构

```
staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/
    │
    ├── generic/
    │   ├── interfaces.go      ← Source, Dispatcher, WebhookInvocation 接口
    │   └── webhook.go         ← Webhook 核心：匹配逻辑 + Dispatch 入口
    │
    ├── mutating/
    │   ├── plugin.go          ← 注册为 "MutatingAdmissionWebhook"
    │   └── dispatcher.go      ← 串行调度，应用 JSON patch
    │
    ├── validating/
    │   ├── plugin.go          ← 注册为 "ValidatingAdmissionWebhook"
    │   └── dispatcher.go      ← 并行调度，收集错误
    │
    ├── accessors.go           ← WebhookAccessor：统一访问配置
    ├── request/               ← 构建 AdmissionReview 请求/响应
    ├── predicates/            ← namespace/object/rule 匹配器
    ├── matchconditions/       ← CEL match condition 评估
    └── config/                ← kubeconfig 加载
```

---

## 核心流程：从 Dispatch 到调用 webhook

`generic/webhook.go:250` 的 `Dispatch` 方法是入口：

```go
func (a *Webhook) Dispatch(ctx, attr, o) error {
    hooks := a.hookSource.Webhooks()                  // 1. 获取所有 webhook 配置
    return a.dispatcher.Dispatch(ctx, attr, o, hooks) // 2. 交给具体 dispatcher
}
```

---

## 匹配逻辑：ShouldCallHook

每个 webhook 在调用前要通过 5 层过滤：

```
ShouldCallHook(hook, attr, ...)
    │
    ├── 1. namespaceMatcher.MatchNamespaceSelector()
    │      → webhook 配置的 namespaceSelector 匹配请求的 namespace？
    │      → 不匹配 → return nil (跳过)
    │
    ├── 2. objectMatcher.MatchObjectSelector()
    │      → webhook 配置的 objectSelector 匹配请求的 object labels？
    │      → 不匹配 → return nil (跳过)
    │
    ├── 3. rules.Matcher.Matches()
    │      → webhook 配置的 rules (apiGroups, resources, operations, scope)
    │      → 匹配请求的 GVR/operation？
    │      → 不匹配 → 尝试 Equivalent 匹配（如果 matchPolicy=Equivalent）
    │
    ├── 4. Equivalent resource 匹配（可选）
    │      → 比如 webhook 注册了 extensions/v1beta1/ingresses
    │      → 请求是 networking.k8s.io/v1/ingresses
    │      → 如果是 Equivalent 策略，也算匹配
    │
    └── 5. matchConditions（CEL 表达式）
           → 用 CEL 表达式做更精细的匹配
           → 不匹配 → return nil (跳过)
           → 表达式出错 → return error（fail closed）
```

---

## Mutating Dispatcher — 串行调度

```
hooks: [webhook-A, webhook-B, webhook-C, ...]
    │
    ▼
for i, hook := range hooks {           ← 串行遍历
    │
    ├── ShouldCallHook()               ← 匹配检查
    ├── 不匹配 → skip
    │
    ├── callAttrMutatingHook()
    │   ├── 构建 AdmissionReview 请求
    │   ├── POST 到 webhook 服务
    │   ├── 收到响应
    │   ├── 如果 Allowed && 有 patch:
    │   │   ├── 解析 JSON Patch
    │   │   ├── 应用到 attr.VersionedObject  ← 修改对象！
    │   │   ├── 标记 Dirty = true
    │   │   └── 调用 Defaulter.Default()     ← 再跑一次默认值
    │   └── 标记 changed = true
    │
    └── 如果 webhook 报错:
        ├── FailurePolicy=Ignore → fail open, 继续下一个
        └── FailurePolicy=Fail   → fail closed, 返回错误
}
```

**关键特点**：
- 串行执行，因为每个 webhook 可能修改对象，下一个需要看到修改后的结果
- 每个 webhook 返回 JSON Patch，应用到对象上

---

## Validating Dispatcher — 并行调度

```
hooks: [webhook-A, webhook-B, webhook-C, ...]
    │
    ▼
Phase 1: 收集匹配的 hooks
    for _, hook := range hooks {
        ShouldCallHook() → 匹配的加入 relevantHooks
    }
    │
    ▼
Phase 2: 并行调用
    wg.Add(len(relevantHooks))
    for i := range relevantHooks {
        go func(invocation) {                    ← 每个 hook 一个 goroutine
            ├── callHook()
            │   ├── 构建 AdmissionReview 请求
            │   ├── POST 到 webhook 服务
            │   ├── 收到响应
            │   └── Allowed? → return nil
            │       Rejected? → errCh <- error
            └── defer wg.Done()
        }(relevantHooks[i])
    }
    wg.Wait()
    close(errCh)
    │
    ▼
Phase 3: 收集结果
    for e := range errCh { errs = append(errs, e) }
    if len(errs) == 0 → return nil (全部通过)
    return errs[0]     → 返回第一个错误（拒绝请求）
```

---

## Mutating vs Validating 对比

| 特性 | Mutating Dispatcher | Validating Dispatcher |
|------|--------------------|-----------------------|
| 执行方式 | 串行（for 循环） | 并行（goroutine + WaitGroup） |
| 能否修改对象 | 能（JSON Patch 应用） | 不能 |
| Reinvocation | 支持 | 不支持 |
| 错误处理 | 按 FailurePolicy 决定 fail open/closed | 同样，但并行收集 |
| 调用时机 | Mutating Admit 阶段 | Validating Validate 阶段 |

---

## 为什么 Mutating 要串行而 Validating 可以并行？

**Mutating 串行的原因**：
```
webhook-A: 给 pod 加 label "app=test"
webhook-B: 检查 pod 有没有 label "app=test"，有的话加 annotation
→ B 必须看到 A 的修改，所以必须串行
```

**Validating 并行的原因**：
```
webhook-C: 检查 image 是否来自可信 registry
webhook-D: 检查 resource limits 是否设置
→ 两者互不影响，各自看到的是同一个最终对象
```

---

## changed 什么时候为 true

```go
changed = !apiequality.Semantic.DeepEqual(attr.VersionedObject, newVersionedObject)
```

webhook 返回的 patch 应用到对象后，与原对象做语义比较，不相等则为 true。

边界情况：webhook 返回了 patch，但 patch 对对象没有实质影响（例如修改了一个字段又改回原值），`changed` 就是 `false`，不会触发后续的重调机制。
