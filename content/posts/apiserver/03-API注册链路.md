---
title: "K8s API Server 源码笔记（三）：API 注册链路"
date: 2026-05-23T03:00:00+08:00
draft: false
tags: ["Kubernetes", "源码分析", "API Server"]
categories: ["云原生"]
series: ["K8s API Server 源码笔记"]
summary: "rest.Storage 接口、APIGroupInfo、APIGroupVersion、APIInstaller、六层拆包注册"
weight: 3
---

## 一句话概括

把"资源的 CRUD 实现"变成"HTTP 路由"。

---

## 全景总览

```
Instance.InstallLegacyAPI                     ← 第1层：组装
  │  构建 APIGroupInfo
  ▼
GenericAPIServer.InstallLegacyAPIGroup        ← 第2层：校验 + 分发
  ▼
GenericAPIServer.installAPIResources          ← 第3层：按版本拆
  │  遍历 PrioritizedVersions，构建 APIGroupVersion
  ▼
APIGroupVersion.InstallREST                   ← 第4层：创建 installer
  │  构建 APIInstaller，调用 Install()
  ▼
APIInstaller.Install                          ← 第5层：按资源拆
  │  遍历 Storage map，逐个调用
  ▼
APIInstaller.registerResourceHandlers         ← 第6层：type assert → 注册路由
```

本质就是一个 map 的三层拆包过程：先按 Group 拆版本，再按版本拆资源，最后按资源的能力注册路由。

---

## 核心结构体

### rest.Storage — 一切的起点

```go
type Storage interface {
    New() runtime.Object  // 创建一个空对象，用来接收请求数据
    Destroy()             // 关闭时清理资源
}
```

这是"资源处理器"的最底层契约。真正的 CRUD 能力靠的是下面这些可选接口：

| 接口 | 对应的 HTTP 操作 | 核心方法 |
|------|-----------------|---------|
| Lister | GET /pods（列表） | `List(ctx, options) → (对象列表, err)` |
| Getter | GET /pods/{name}（单个） | `Get(ctx, name, options) → (对象, err)` |
| Creater | POST /pods（创建） | `Create(ctx, obj, ...) → (对象, err)` |
| Updater | PUT /pods/{name}（更新） | `Update(ctx, name, objInfo, ...) → (对象, bool, err)` |
| GracefulDeleter | DELETE /pods/{name}（删除） | `Delete(ctx, name, ...) → (对象, bool, err)` |
| Watcher | GET /pods?watch=true（监听） | `Watch(ctx, options) → watch.Interface` |
| CollectionDeleter | DELETE /pods（批量删除） | `DeleteCollection(ctx, ...) → (对象, err)` |
| Connecter | GET /pods/{name}/exec（连接） | `Connect(ctx, id, ...) → http.Handler` |
| Patcher | PATCH /pods/{name}（补丁） | 组合了 Getter + Updater |

**设计精髓**：`registerResourceHandlers` 里做的事情就是 type assert — 拿着你的 Storage 去问"你是不是 Lister？是不是 Getter？"，实现了哪些就注册哪些路由。

---

### APIGroupInfo — 一个 API 分组的注册表

```go
type APIGroupInfo struct {
    PrioritizedVersions          []schema.GroupVersion              // 支持的版本，按优先级排
    VersionedResourcesStorageMap map[string]map[string]rest.Storage // 核心！版本→资源名→Storage
    Scheme                       *runtime.Scheme                    // 类型注册表
    NegotiatedSerializer         runtime.NegotiatedSerializer       // JSON/Protobuf 编解码
    ParameterCodec               runtime.ParameterCodec             // URL 参数解析
    MetaGroupVersion             schema.GroupVersion                // meta.k8s.io/v1
    StaticOpenAPISpec            map[string]*spec.Schema            // OpenAPI schema
}
```

核心字段 `VersionedResourcesStorageMap` 的结构：

```
VersionedResourcesStorageMap = {
    "v1": {                          ← 版本
        "pods":     &PodStorage{},   ← 资源名 → 处理器
        "services": &ServiceStorage{},
        "nodes":    &NodeStorage{},
    }
}
```

**类比**：如果把 API Server 比作一个商场，APIGroupInfo 就是一张"楼层导览图" — 告诉你每一层（版本）有哪些店铺（资源），每个店铺的老板是谁（Storage）。

---

### APIGroupVersion — 一个版本的执行上下文

```go
type APIGroupVersion struct {
    Storage      map[string]rest.Storage  // 这个版本下的资源→Storage 映射
    Root         string                   // URL 根路径，如 "/api"
    GroupVersion schema.GroupVersion      // 如 "v1"
    Serializer   runtime.NegotiatedSerializer
    Typer        runtime.ObjectTyper      // 判断对象的 GVK
    Creater      runtime.ObjectCreater    // 按 GVK 创建空对象
    Authorizer   authorizer.Authorizer    // 鉴权
    Admit        admission.Interface      // 准入控制
}
```

和 APIGroupInfo 的区别：
- **APIGroupInfo** 是所有版本的合集（输入）
- **APIGroupVersion** 是单个版本的执行上下文（中间产物）

---

### APIInstaller — 单个版本的安装工人

```go
type APIInstaller struct {
    group             *APIGroupVersion  // 携带版本上下文
    prefix            string            // URL 前缀，如 "/api/v1"
    minRequestTimeout time.Duration
}
```

极简的"工具人"结构体，拿着一个 APIGroupVersion 和一个 URL 前缀，遍历 Storage map，逐个调用 `registerResourceHandlers`。

---

## 结构体之间的关系

```
APIGroupInfo          "我管着整个 API Group，下面有多个版本"
    │
    │ installAPIResources 拆版本
    ▼
APIGroupVersion       "我是其中一个版本，带着这个版本的 Storage map 和所有配置"
    │
    │ InstallREST 创建 installer
    ▼
APIInstaller          "我是这个版本的安装工人，拿着前缀和 Storage map 去注册路由"
    │
    │ Install() 遍历每个资源
    ▼
registerResourceHandlers  "我拿着单个 Storage，type assert 看它实现了哪些接口，
                          就注册哪些 HTTP 路由"
```

---

## 六层链路详解

### 第1层：Instance.InstallLegacyAPI

组装 APIGroupInfo。`NewLegacyRESTStorage` 内部创建每个资源的 Storage（PodStorage、ServiceStorage、NodeStorage...），然后组装成：

```go
apiGroupInfo.VersionedResourcesStorageMap = map[string]map[string]rest.Storage{
    "v1": {
        "pods":                   &PodStorage{},
        "services":               &ServiceStorage{},
        "nodes":                  &NodeStorage{},
        "endpoints":              &EndpointStorage{},
        "configmaps":             &ConfigMapStorage{},
        // ... 几十个资源
    }
}
```

交给下一步：
```go
m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo)
//                                         ↑ "/api"                    ↑ 所有资源的 map
```

### 第2层：GenericAPIServer.InstallLegacyAPIGroup

校验 + 转发 + 挂 discovery handler。

```go
func (s *GenericAPIServer) InstallLegacyAPIGroup(apiPrefix string, apiGroupInfo *APIGroupInfo) error {
    // 1. 校验前缀是否合法
    if !s.legacyAPIGroupPrefixes.Has(apiPrefix) { return fmt.Errorf(...) }
    // 2. 拿 OpenAPI 模型
    openAPIModels, err := s.getOpenAPIModels(apiPrefix, apiGroupInfo)
    // 3. 核心：转发给 installAPIResources
    if err := s.installAPIResources(apiPrefix, apiGroupInfo, openAPIModels); err != nil { return err }
    // 4. 在 /api 挂一个 discovery handler
    legacyRootAPIHandler := discovery.NewLegacyRootAPIHandler(...)
    s.Handler.GoRestfulContainer.Add(legacyRootAPIHandler.WebService())
}
```

### 第3层：GenericAPIServer.installAPIResources

第一次拆包 — 从"所有版本"拆出"单个版本"。

```go
for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
    apiGroupVersion, err := s.getAPIGroupVersion(apiGroupInfo, groupVersion, apiPrefix)
    discoveryAPIResources, r, err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer)
}
```

### 第4层：APIGroupVersion.InstallREST

构建 APIInstaller，拼出 URL 前缀，调用 Install()。

```go
func (g *APIGroupVersion) InstallREST(container *restful.Container) (...) {
    prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
    // 对于 legacy API: "/api" + "" + "v1" = "/api/v1"
    installer := &APIInstaller{group: g, prefix: prefix}
    apiResources, resourceInfos, ws, registrationErrors := installer.Install()
    // 挂版本级 discovery handler
    versionDiscoveryHandler := discovery.NewAPIVersionHandler(...)
    versionDiscoveryHandler.AddToWebService(ws)
    container.Add(ws)
}
```

### 第5层：APIInstaller.Install

第二次拆包 — 从"所有资源"拆出"单个资源"。

```go
func (a *APIInstaller) Install() ([]metav1.APIResource, []*storageversion.ResourceInfo, *restful.WebService, []error) {
    ws := a.newWebService()
    paths := make([]string, len(a.group.Storage))
    for path := range a.group.Storage { paths[i] = path }
    sort.Strings(paths)
    for _, path := range paths {
        apiResource, resourceInfo, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
    }
    return apiResources, resourceInfos, ws, errors
}
```

### 第6层：registerResourceHandlers — 整条链路的终点

**步骤一：探测 Storage 实现了哪些接口**

```go
creater, isCreater := storage.(rest.Creater)
lister, isLister := storage.(rest.Lister)
getter, isGetter := storage.(rest.Getter)
gracefulDeleter, isGracefulDeleter := storage.(rest.GracefulDeleter)
// ...
```

**步骤二：根据探测结果构建 actions 列表**

```go
actions = appendIf(actions, action{"LIST",             resourcePath,           ...}, isLister)
actions = appendIf(actions, action{"POST",             resourcePath,           ...}, isCreater)
actions = appendIf(actions, action{"DELETECOLLECTION",  resourcePath,           ...}, isCollectionDeleter)
actions = appendIf(actions, action{"GET",              itemPath,               ...}, isGetter)
actions = appendIf(actions, action{"PUT",              itemPath,               ...}, isUpdater)
actions = appendIf(actions, action{"PATCH",            itemPath,               ...}, isPatcher)
actions = appendIf(actions, action{"DELETE",           itemPath,               ...}, isGracefulDeleter)
actions = appendIf(actions, action{"CONNECT",          itemPath,               ...}, isConnecter)
```

**步骤三：遍历 actions，注册 go-restful 路由**

```go
for _, action := range actions {
    switch action.Verb {
    case "GET":
        route := ws.GET(action.Path).To(handler)...
    case "POST":
        route := ws.POST(action.Path).To(handler)...
    // ...
    }
    ws.Route(route)
}
```

---

## InstallLegacyAPI vs InstallAPIs

| | InstallLegacyAPI | InstallAPIs |
|---|---|---|
| 入口数量 | 只处理一个 group (core/v1) | 批量处理多个 group |
| 调用 | `InstallLegacyAPIGroup` | `InstallAPIGroups` |
| URL 前缀 | `/api/v1/pods`（无 group 名） | `/apis/apps/v1/deployments`（有 group 名） |
| discovery | 在 `/api` 挂版本发现 | 在 `/apis/{group}` 挂 group 发现 |
| 额外步骤 | 无 | 多了过期资源检查（`RemoveDeletedKinds`） |

底层最终都走到同一个 `installAPIResources`，两条路完全一样。

---

## OpenAPI

在 `APIGroupInfo` 中对应 `StaticOpenAPISpec` 字段。它的产生过程：

1. `registerResourceHandlers` 注册路由时，同时把每个资源的 Go struct 转成 OpenAPI schema
2. `InstallREST` 完成后，所有资源的 schema 汇总成 `StaticOpenAPISpec`

用途：kubectl 自动补全、`kubectl explain`、Server-Side Apply 字段校验、Swagger UI、客户端代码生成、第三方集成。

**一句话总结**：注册链路不仅注册了 HTTP 路由，同时生成了对应的 OpenAPI 文档，两者是同步产生的。
