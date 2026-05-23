---
title: "K8s API Server 源码笔记（五）：PathRecorderMux 路由器"
date: 2026-05-23T05:00:00+08:00
draft: false
tags: ["Kubernetes", "源码分析", "API Server"]
categories: ["云原生"]
series: ["K8s API Server 源码笔记"]
summary: "精准匹配 + 最长前缀匹配、前缀排序、线程安全、路径堆栈记录"
weight: 5
---

## 概述

`PathRecorderMux` 是 Kubernetes API Server 中处理 `/healthz`、`/metrics` 等非 go-restful 路由的自定义 HTTP 路由多路复用器。

它是一个**带路径记录功能、线程安全、支持精准匹配 + 前缀匹配的增强版 HTTP 路由器**。

源码位置：`vendor/k8s.io/apiserver/pkg/server/mux/pathrecorder.go`

---

## PathRecorderMux 结构体

```go
type PathRecorderMux struct {
    name            string                    // 日志标识名
    lock            sync.Mutex                // 互斥锁，保证并发安全
    notFoundHandler http.Handler              // 自定义 404 处理器
    pathToHandler   map[string]http.Handler   // 精准路径匹配表
    prefixToHandler map[string]http.Handler   // 路径前缀匹配表
    mux             atomic.Value              // 原子存储的底层路由器
    exposedPaths    []string                  // 对外暴露的路径列表
    pathStacks      map[string]string         // 路径注册堆栈（调试用）
}
```

| 字段 | 作用 |
|------|------|
| `name` | 给路由器起名，用于日志追踪 |
| `lock` | 互斥锁，多 goroutine 并发注册路径时保证线程安全 |
| `notFoundHandler` | 请求路径未注册时的自定义 404 处理器 |
| `pathToHandler` | 精准匹配表：请求路径**完全等于**注册路径才命中 |
| `prefixToHandler` | 前缀匹配表：请求路径**以注册前缀开头**就命中 |
| `mux` | `atomic.Value`，存储真正负责请求分发的底层路由器 |
| `exposedPaths` | 所有对外提供服务的 API 路径 |
| `pathStacks` | 记录每个路径的注册代码堆栈，重复注册时告诉你是谁先注册的 |

---

## pathHandler — 真正干活的执行引擎

```go
type pathHandler struct {
    muxName         string
    pathToHandler   map[string]http.Handler  // 精准匹配表
    prefixHandlers  []prefixHandler          // 前缀匹配列表（有序！）
    notFoundHandler http.Handler             // 404 处理器
}
```

**执行规则**：精准匹配优先 → 最长前缀匹配次之 → 最后 404

```go
func (h *pathHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 1. 先精准匹配
    if exactHandler, ok := h.pathToHandler[r.URL.Path]; ok {
        exactHandler.ServeHTTP(w, r)
        return
    }
    // 2. 再按"最长前缀优先"匹配
    for _, prefixHandler := range h.prefixHandlers {
        if strings.HasPrefix(r.URL.Path, prefixHandler.prefix) {
            prefixHandler.handler.ServeHTTP(w, r)
            return
        }
    }
    // 3. 最后 404
    h.notFoundHandler.ServeHTTP(w, r)
}
```

---

## 前缀排序规则

`prefixHandlers` 是切片（有序列表），不是 map。前缀匹配必须按"最长、最精确"优先：

排序优先级：斜杠越多越优先 → 长度越长越优先 → 字典序

示例匹配顺序：`/api/v1/user/` > `/api/v1/` > `/api/`

---

## 核心方法

### 精准匹配注册

```go
func (m *PathRecorderMux) Handle(path string, handler http.Handler) {
    m.lock.Lock()
    defer m.lock.Unlock()
    m.trackCallers(path)                          // 记录注册堆栈
    m.exposedPaths = append(m.exposedPaths, path) // 加入公开路径
    m.pathToHandler[path] = handler
    m.refreshMuxLocked()                          // 刷新路由表
}
```

### 不公开精准匹配

```go
func (m *PathRecorderMux) UnlistedHandle(...) {
    // 一样，但不加入 exposedPaths — 隐藏接口
}
```

### 前缀匹配注册

```go
func (m *PathRecorderMux) HandlePrefix(path string, handler http.Handler) {
    if !strings.HasSuffix(path, "/") {
        panic("must end in a trailing slash")  // 强制要求以 / 结尾
    }
    m.prefixToHandler[path] = handler
    m.refreshMuxLocked()
}
```

### 刷新路由表

```go
func (m *PathRecorderMux) refreshMuxLocked() {
    newMux := &pathHandler{...}
    // 拷贝精准匹配
    for path, handler := range m.pathToHandler {
        newMux.pathToHandler[path] = handler
    }
    // 排序前缀（最长优先）
    keys := sets.StringKeySet(m.prefixToHandler).List()
    sort.Sort(sort.Reverse(byPrefixPriority(keys)))
    // 组装前缀
    for _, prefix := range keys {
        newMux.prefixHandlers = append(newMux.prefixHandlers, prefixHandler{prefix, handler})
    }
    // 原子替换（无锁高性能）
    m.mux.Store(newMux)
}
```

每次注册/删除路径都会重建全新路由表，用 `atomic.Store` 无锁更新。

### 报错友好的堆栈记录

```go
func (m *PathRecorderMux) trackCallers(path string) {
    stack := string(debug.Stack())
    if existingStack, ok := m.pathStacks[path]; ok {
        utilruntime.HandleError(fmt.Errorf("duplicate path registration..."))
    }
    m.pathStacks[path] = stack
}
```

原生 Go mux 重复注册同一个路径时只提示 `multiple registrations for /xxx`，这个字段记录了第一次是在哪里注册的，秒定位 bug。

---

## 总结

1. `PathRecorderMux` = 带记录、报错友好、线程安全的增强路由
2. `pathHandler` = 真正执行请求的执行器
3. **匹配规则**：精准匹配 → 最长前缀匹配 → 404
4. **前缀必须排序**：斜杠多 > 长度长，保证最精确优先
5. **trackCallers**：记录注册位置，解决重复注册报错不清晰
6. **refreshMuxLocked**：每次注册都重建路由表 + 原子替换，高并发安全
