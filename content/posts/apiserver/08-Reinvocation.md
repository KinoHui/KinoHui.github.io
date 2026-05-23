---
title: "K8s API Server 源码笔记（八）：Reinvocation 机制"
date: 2026-05-23T08:00:00+08:00
draft: false
tags: ["Kubernetes", "源码分析", "API Server"]
categories: ["云原生"]
series: ["K8s API Server 源码笔记"]
summary: "两轮执行、三层协作、IfIfNeeded/Never 策略、重调触发条件"
weight: 8
---

## 问题场景

```
第一轮 Admit 执行链:
    [LimitRanger] → [ServiceAccount] → [MutatingWebhook] → [ResourceQuota]
                                                │
    LimitRanger 设置了 default resource limits
    ServiceAccount 注入了 serviceAccountName
    MutatingWebhook-A 设置了 label "app=test"
    MutatingWebhook-B 根据 label "app=test" 添加了 annotation
                                                │
                                                ▼
    问题: 如果 MutatingWebhook-A 之后，in-tree 插件又修改了对象怎么办？
    → webhook-B 看到的对象已经不是它最初看到的了
    → 但 B 不知道自己需要重新执行
```

---

## 解决方案：两轮执行

```
第一轮: chain.Admit()  (所有插件顺序执行)
    │
    ├── LimitRanger.Admit()         → 设置 default limits
    ├── ServiceAccount.Admit()      → 注入 SA
    ├── MutatingWebhook.Admit()     → webhook-A 设置 label, webhook-B 添加 annotation
    │     └── webhook-A: ReinvocationPolicy=IfNeeded → 记录到可重调列表
    │         webhook-B: ReinvocationPolicy=Never    → 不记录
    │
    ├── 对象变了 → reinvokeCtx.SetShouldReinvoke()
    │
    ▼
reinvoker 检查: ShouldReinvoke() == true
    │
    ▼
第二轮: chain.Admit()  (所有插件再执行一遍)
    │
    ├── LimitRanger.Admit()         → 执行（但发现对象没变，跳过实际修改）
    ├── ServiceAccount.Admit()      → 同上
    ├── MutatingWebhook.Admit()     → 根据 reinvoke 状态决定是否调用
    │     └── webhook-A: ShouldReinvokeWebhook("A") == true  → 重新调用
    │         webhook-B: ShouldReinvokeWebhook("B") == false → 跳过
    │
    ▼
  完成
```

---

## 三层协作

### Layer 1: reinvoker（reinvocation.go）

```go
func (r *reinvoker) Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error {
    // 第一轮：执行整个 mutating admission chain
    err := mutator.Admit(ctx, a, o)
    if err != nil { return err }

    s := a.GetReinvocationContext()
    if s.ShouldReinvoke() {
        s.SetIsReinvoke()
        // 第二轮：再跑一遍整个 mutating admission chain
        return mutator.Admit(ctx, a, o)
    }
    return nil
}
```

**关键**：reinvoke 不是只重跑 webhook，而是重跑整个 admission mutating chain。

### Layer 2: reinvocationContext

```go
type reinvocationContext struct {
    isReinvoke       bool                   // 当前是否在第二轮
    reinvokeRequested bool                  // 是否需要第二轮
    values map[string]interface{}           // 每个插件的私有状态
}
```

### Layer 3: webhookReinvokeContext

```go
type webhookReinvokeContext struct {
    lastWebhookOutput runtime.Object            // 上一轮 webhook 输出的快照
    previouslyInvokedReinvocableWebhooks sets.String
    reinvokeWebhooks sets.String                // 需要重调的 webhook UID
}
```

核心方法：
- `IsOutputChangedSinceLastWebhookInvocation(obj)` → 比较当前对象和 `lastWebhookOutput` 是否不同
- `AddReinvocableWebhookToPreviouslyInvoked(uid)` → 标记该 webhook 配置了 `IfNeededReinvocationPolicy`
- `RequireReinvokingPreviouslyInvokedPlugins()` → 把所有 previouslyInvoked 中的 webhook 加入 reinvokeWebhooks

---

## 关键代码流程

### 第一轮执行时（IsReinvoke() == false）

```go
for i, hook := range hooks {
    invocation, statusErr := a.plugin.ShouldCallHook(...)
    if invocation == nil { continue }

    // 正常调用
    changed, err := a.callAttrMutatingHook(...)

    // 如果 webhook 改了对象
    if changed {
        webhookReinvokeCtx.RequireReinvokingPreviouslyInvokedPlugins()
        reinvokeCtx.SetShouldReinvoke()
    }

    // 如果这个 webhook 配置了 IfNeeded
    if hook.ReinvocationPolicy == IfNeededReinvocationPolicy {
        webhookReinvokeCtx.AddReinvocableWebhookToPreviouslyInvoked(uid)
    }
}
```

### 第二轮执行时（IsReinvoke() == true）

```go
if reinvokeCtx.IsReinvoke() &&
   webhookReinvokeCtx.IsOutputChangedSinceLastWebhookInvocation(attr.GetObject()) {
    // 对象被 in-tree 插件改了 → 标记所有可重调 webhook 需要执行
    webhookReinvokeCtx.RequireReinvokingPreviouslyInvokedPlugins()
}

for i, hook := range hooks {
    invocation, _ := a.plugin.ShouldCallHook(...)

    // 关键过滤
    if reinvokeCtx.IsReinvoke() &&
       !webhookReinvokeCtx.ShouldReinvokeWebhook(invocation.Webhook.GetUID()) {
        continue  // 这个 webhook 不需要重调，跳过
    }

    // 执行重调
    changed, err := a.callAttrMutatingHook(...)
}
```

---

## 重调触发的两个来源

### 来源 1：webhook 修改了对象

当某个 webhook 的 patch 改变了对象时：

```go
if changed {
    webhookReinvokeCtx.RequireReinvokingPreviouslyInvokedPlugins()
    reinvokeCtx.SetShouldReinvoke()  // 唯一的调用点
}
```

**`SetShouldReinvoke()` 的唯一调用点在 webhook dispatcher 内部**。没有任何 in-tree 插件调用它。

### 来源 2：in-tree 插件修改了对象

第二轮中检测到对象变化：

```go
if reinvokeCtx.IsReinvoke() &&
   webhookReinvokeCtx.IsOutputChangedSinceLastWebhookInvocation(attr.GetObject()) {
    webhookReinvokeCtx.RequireReinvokingPreviouslyInvokedPlugins()
}
```

---

## 被重调的条件

被重调的 webhook 必须**同时满足两个条件**：

1. `ReinvocationPolicy` 为 `IfNeeded`
2. 在它之后有其他 webhook（或 in-tree 插件）修改了对象

如果一个 `IfNeeded` webhook 执行后，后续没有 webhook 改变对象，它就不会被重调 — 因为没有必要。这也正是 `IfNeeded` 语义的体现："仅在需要时重调"。

---

## ReinvocationPolicy 的两种值

| 策略 | 含义 | 何时重调 |
|------|------|---------|
| Never（默认） | 永不重调 | 第二轮被跳过 |
| IfNeeded | 需要时重调 | 当对象被后续插件修改时，第二轮会重新调用 |

---

## 设计要点总结

1. **触发条件**：任何一个 mutating webhook 改变了对象 → `SetShouldReinvoke()`
2. **重调范围**：只有配置了 `IfNeededReinvocationPolicy` 的 webhook 才会被重调
3. **第二轮判断**：`webhookReinvokeContext` 跟踪对象快照，如果 in-tree 插件在第二轮又改了对象，会再次触发可重调 webhook 的执行
4. **默认安全**：`Never` 策略的 webhook 永远不会被重调，避免不必要的副作用
5. **链式传播**：如果 webhook-A 在第二轮改了对象，之前标记为 `IfNeeded` 的 webhook 也会被再次重调
