# Appsmith 源码分析（一）：整体架构总览

> 本文是系列第一篇：仓库分层、前端/后端架构、CE/EE 共存机制、域模型关系、前后端交互、编辑器与运行时（view 模式）的区别。
> 专题深入见后续四篇：[02-页面保存](02-page-save.md) · [03-组件抽象](03-widget-abstraction.md) · [04-属性配置](04-property-config.md) · [05-数据绑定](05-data-binding.md)。
> 所有结论均标注代码位置（`文件路径:行号`），行号基于当前 release 分支。

---

## 1. 项目是什么

**Appsmith 是一个开源低代码平台**：用户在浏览器编辑器里拖拽组件（Widget）、配置属性、绑定数据（API/Query/JS 对象），发布后供最终用户使用。

```
┌────────────────────────── app/client（React + Redux）──────────────────────────┐
│  编辑器（AppIDE）+ 查看器（AppViewer）                                          │
│  widgets（~70 种组件）· layoutSystems（3 套布局）· workers（eval 求值引擎）     │
└───────────────▲─────────────────────────────────────────────▲─────────────────┘
                │ REST /api/v1/*（axios）                      │ 首屏合并加载 /consolidated-api
┌───────────────┴─────────────────────────────────────────────┴─────────────────┐
│  app/server（Java Spring Boot WebFlux 响应式，Maven 多模块）                   │
│  appsmith-server（主服务）· appsmith-interfaces（插件契约）                    │
│  appsmith-plugins（26 个数据源插件）· appsmith-git · reactive-caching(Redis)   │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
                     MongoDB（newPage 集合内嵌页面 DSL）
                     RTS（Node，packages/rts：AST/DSL/git 服务）
```

---

## 2. 仓库顶层结构

| 目录 | 职责 |
|---|---|
| `app/client` | React+Redux 前端（编辑器 + 查看器），CRA 风格 webpack 自定义构建（`app/client/config/webpack.config.js`），yarn workspaces 管理 `packages/**` 子包 |
| `app/server` | Java Spring Boot（WebFlux 响应式）后端，Maven 多模块：`appsmith-server`（主服务）、`appsmith-interfaces`（外部模型/插件契约，包名 `com.appsmith.external`）、`appsmith-plugins`（26 个数据源插件：postgres/rest/mongo/openAi 等）、`appsmith-git`（git 工具）、`reactive-caching`（Redis 缓存/分布式锁） |
| `app/util` | 仅 `is_wsl.sh`/`is_wsl_test.sh`，容器启动时检测 WSL 环境 |
| `deploy` | 部署编排：`docker/`（docker-compose、base.dockerfile）、`helm/`（K8s chart）、`ansible/`、`aws/`、`heroku/`、`packer/` 等 |
| `scripts` | CI/发布辅助脚本（`prepare_server_artifacts.sh`、`deploy_preview.sh`、`health_check.sh`、`local_testing.sh` 等） |
| 其他 | `contributions/`（贡献者文档）、`static/`（静态图）、`docs/`；根 `Dockerfile` 组装单容器产物（`COPY ./app/client/build editor/`、`COPY ./app/client/packages/rts/dist rts/`，`Dockerfile:30-37`），内嵌 Mongo（`server/mongo/server.jar`） |

---

## 3. 前端架构（app/client）

### 3.1 入口与启动

入口 `app/client/src/index.tsx`：

- `import "./preload-route-chunks"` 预加载路由分块（:2）
- `import "utils/workerInstances"` 初始化 **eval worker**（:4）
- `import "widgets"` 注册 widget loader（:8）
- `store` 来自 `./store`，AppRouter 来自 **`ee/AppRouter`**（:21）—— **组合根固定走 ee 路径**
- `runSagaMiddleware()`（:35）+ `appInitializer()`（:37），Faro/Sentry 埋点，ReactDOM.render（:82）

### 3.2 路由组织

- 路由常量：`app/client/src/ce/constants/routes/appRoutes.ts`
  - `BUILDER_PATH = "/:applicationSlug/:pageSlug(.*-):basePageId.../edit"`（:21）、`VIEWER_PATH`（:23）、自定义 slug 变体 `*_CUSTOM_PATH`（:22,24）、静态 slug 路由 `*_STATIC`（:27-28）
- 顶层路由表：`app/client/src/ce/AppRouter.tsx:78-146`（`<Switch>` 内 SentryRoute）：Workspace、Applications、UserAuth、Templates、AdminSettings、**`AppIDE`**（BUILDER_PATH，:133-134）、**`AppViewerLoader`**（VIEWER_PATH，:135-136）、PageNotFound

### 3.3 IDE 组织（新版编辑器）

- `src/IDE/`：与具体产品无关的 IDE 基础组件库（`IDE/index.ts:12-55`）：`IDEToolbar`（Structure/Toolbar）、`IDEBottomView`（可缩放底部面板）、`IDESidePaneWrapper`、`EditableName`、`ViewHideBehaviour` 等
- `src/pages/AppIDE/AppIDE.tsx`：编辑器页面组件。`componentDidMount` 里 `urlBuilder.setCurrentBasePageId()` + `editorInitializer()` 构建 widget 配置树（:86-90）；页面/分支切换时重新 `initEditor({mode: APP_MODE.EDIT})` 或 `updateCurrentPage+setupPage`（:194-215）；渲染 `<GitApplicationContextProvider><GlobalHotKeys><IDE/>`（:245-250）
- IDE 布局：`src/pages/AppIDE/layouts/index.tsx:20-28` 按 feature flag `release_ide_animations_enabled` 切换 `AnimatedLayout` / `StaticLayout`
- `StaticLayout.tsx` 组成 `Sidebar / LeftPane / MainPane / RightPane`（:57-67）：
  - `routers/Sidebar.tsx`：`IDESidebar`（@appsmith/ads），切换 DATA/EDITOR/SETTINGS 状态
  - `routers/LeftPane.tsx`：按子路由渲染 DataSidePane（datasources）、LibrarySidePane、TriggerSettingsPane、AppSettingsPane
  - `routers/MainPane/MainPane.tsx:16-32`：预览模式直接渲染 `WidgetsEditor`，否则走 `MainPaneRoutes(path)`（`ce/pages/AppIDE/layout/routers/MainPane/constants.ts:40-110`，把 Canvas/API/Query/JS/SAAS/设置等子路由映射回 WidgetsEditor 或对应编辑器）
  - `SegmentSwitcher.tsx:12-30`：Queries/JS/UI 三段切换
- `EditorState` 枚举：`src/IDE/enums.ts:7-12`（DATA/EDITOR/SETTINGS/TRIGGER_SETTINGS）
- `src/PluginActionEditor/`：查询/API 编辑器的可复用上下文与组件（PluginActionForm、Settings、Response 等），被 AppIDE 的 QueryEditor 路由消费

### 3.4 Redux 组织

- `src/store.ts:30-38`：`createStore(appReducer)`，中间件顺序 `packageMiddleware → sagaMiddleware → routeParamsMiddleware`，附加 `reduxBatch`、Sentry ReduxEnhancer；`runSagaMiddleware()` 运行 `rootSaga`（:53）
- **所有导入都走 `ee/*` 别名**：`appReducer from "ee/reducers"`、`rootSaga from "ee/sagas"`（store.ts:4-6）
- reducer 组合根：`ce/reducers/index.tsx:94-104`，顶层键：

```
entities / ui / evaluations / form(redux-form) / settings / organization / linting / git / aiAssistant
```

- reducer 分层：`src/reducers/{entityReducers, uiReducers, evaluationReducers, lintingReducers}` 是共享主体（uiReducers 约 45 个：editor、propertyPane、appView、gitSync、theme、debugger…；entityReducers：canvasWidgets、pageList、plugins、datasource、meta/metaWidgets、actions、jsActions…），`ce/reducers/` 只放少量 CE 专属（applicationsReducer、workspaceReducer 等），`ee/reducers/*` 再 combineReducers 包装 ce 并可注入 EE 专属 reducer（`ee/reducers/entityReducers/index.ts`、`ee/reducers/uiReducers/index.tsx`）
- saga：`ce/sagas/index.tsx:65-125` 注册约 60 个 saga（initSagas、pageSagas、actionSagas、evaluationsSaga、widgetOperationSagas、gitSyncSagas…），`ee/sagas/index.tsx:12-37` 的 `rootSaga` 用 `race(SAFE_CRASH_APPSMITH)` 实现"safe crash 后自动重启全部 saga"

### 3.5 CE/EE 共存机制（前端）⭐

本仓库是 CE 开源发布版，但保留了完整的 EE 分层骨架：

- 官方说明：`app/client/src/enterprise/README.md`（"用 `ee` 别名切换 CE/EE 文件以减少合并冲突；CE 内消费者统一 import `ee/...` 路径"）
- 两种形态：
  1. **纯转发 shim**：如 `ee/AppRouter.tsx:1-3`（`export default CE_AppRouter`）、`ee/constants/ReduxActionConstants.tsx:1`（`export * from "ce/..."`）、`ee/reducers/index.tsx`
  2. **类扩展空壳**：如 `ee/api/ApplicationApi.tsx`（`class ApplicationApi extends CE_ApplicationApi {}`）、`ee/sagas/ApplicationSagas.tsx` —— EE 闭源仓库会覆写同名文件注入企业逻辑
- 反向约束：个别模块标注"CE-owned，禁止加 ee shim"（如 `ce/reducers/aiAssistantReducer` 的 eslint 注释，`ce/reducers/index.tsx:89-92`）
- 目录镜像：`src/ce/` 与 `src/ee/` 子目录结构完全对应（AppRouter、pages、reducers、sagas、api、constants、selectors、middlewares、workers…）；`src/enterprise/` 在 CE 中只剩 README

> **搜索技巧**：若跳转落到 `ee/` 的转发壳，需跳到 `ce/` 同路径找实现。

---

## 4. 后端架构（app/server）

### 4.1 Spring 模块划分

- 启动类：`appsmith-server/.../ServerApplication.java:12`（`@ComponentScan({"com.appsmith"})`，WebFlux 响应式；配置链 `application.properties → application-ee.properties → application-ce.properties`，`src/main/resources/application.properties:1`）
- `appsmith-interfaces`：`com.appsmith.external.*` —— 插件契约 `PluginExecutor`（`PluginExecutor.java:37`）、共享域模型 `BaseDomain`（id/policies/createdAt）、`Datasource`、`ActionDTO`、`DatasourceStorage`（datasourceId+environmentId 双键）
- `appsmith-plugins/*`：每个数据源一个 Maven 模块，实现 `PluginExecutor<C>`（如 `restApiPlugin/.../RestApiPlugin.java:46-53` 内嵌 `RestApiPluginExecutor`）
- `appsmith-git`：`com.appsmith.git.{configurations,constants,converters,dto,files,handler,helpers,service}` —— git 序列化/执行
- `reactive-caching`：`@Cache/@CacheEvict/@DistributedLock` 注解 + Redis CacheManager（Reactive）
- 主包按 **"域 → {base, controllers, services, repositories, 子域}"** 组织，顶层包：`applications, newpages, newactions, actioncollections, layouts, datasources, datasourcestorages, plugins, git, themes, refactors, clonepage, fork, imports, exports, publish, onload, staticurl, searchentities, featureflags, acl, cron, migrations` 等
- **Controller 层**（`server/controllers/`，`@RequestMapping(Url.X)`）：ApplicationController、PageController、LayoutController、ActionController、ActionCollectionController、DatasourceController、PluginController、ThemeController、GitController、**ConsolidatedAPIController** 等；每个 `XxxController extends XxxControllerCE`（EE 挂载点，如 `LayoutController.java:15`）
- **REST 路径常量**：`constants/ce/UrlCE.java:6-35` —— `/api/v1/{applications|pages|layouts|actions|collections/actions|datasources|plugins|themes|git|consolidated-api|...}`
- **Repository 层**（Spring Data Reactive MongoDB）：`repositories/` 每域 `XxxRepository + CustomXxxRepository(Impl)`（自定义聚合查询）
- **Service 层**：`services/` + `services/ce/`（XxxServiceCE/Impl）+ `services/ce_compatible/`；如 `ApplicationPageService`（应用/页面生命周期、publish）、`NewPageServiceCEImpl`、`ConsolidatedAPIServiceCEImpl`、`LayoutActionService`（DSL 保存与 on-load 依赖计算）、`DatasourceContextService`（连接池上下文）
- 服务端同样 CE/EE 分层：域类 `domains/ce/ApplicationCE.java`（字段全集）← `domains/Application.java:21`（`extends ApplicationCE implements Artifact`）；本 CE 版无任何 `ee` Java 目录

### 4.2 关键域对象与关系 ⭐

```
Workspace (workspaceId)
  └─ Application (Mongo, extends ApplicationCE)
       ├─ List<ApplicationPage> pages / publishedPages     ← 发布/草稿双轨（ApplicationCE.java:61-65）
       ├─ unpublishedApplicationDetail / publishedApplicationDetail（:82-86）
       ├─ gitApplicationMetadata (GitArtifactMetadata, :113)、evaluationVersion(:121)
       │ 1..N（applicationId 外键，NewPage.java:23）
       ▼
NewPage (Context, RefAwareDomain)
  ├─ unpublishedPage: PageDTO / publishedPage: PageDTO      ← 同一双轨模式（NewPage.java:25-29）
  └─ PageDTO.layouts[0] = Layout
       ├─ dsl: JSONObject / publishedDsl: JSONObject        ← 页面 DSL 存这里（Layout.java:41-45）
       ├─ layoutOnLoadActions: List<Set<DslExecutableDTO>> （:47-48）+ errors
       └─ getDsl() 按 viewMode 返回 publishedDsl 或 dsl（:108-111）

NewAction (extends NewActionCE, RefAwareDomain)
  ├─ applicationId / workspaceId / pluginId / pluginType（NewActionCE.java:26-36）
  ├─ unpublishedAction: ActionDTO / publishedAction: ActionDTO（:42-46）
  └─ ActionDTO 含 pageId、datasource（引用 Datasource.id）、actionConfiguration、
     fullyQualifiedName、executeOnLoad/runBehaviour、collectionId（JS 对象归属）

Datasource（appsmith-interfaces/.../Datasource.java:28，extends GitSyncedDomain）
  └─ 1..N DatasourceStorage（datasourceId+environmentId，按环境存配置）
     Plugin（domains/Plugin.java:25-39）决定用哪个 appsmith-plugins 执行器
```

通用底座：

- `BaseDomain`：id、policies/policyMap（**ACL 权限就存在每个文档上**）、createdAt/updatedAt
- `RefAwareDomain`：baseId + refType/refName —— **支撑 git 分支复制资源**：branchedPageId → basePageId（`RefAwareDomain.java:13-45`）。这解释了 Controller 里 `@PathVariable branchedPageId` 与前端 URL 里的 `basePageId`
- `Artifact/ArtifactCE/Context` 抽象：Application、Package、Workflow 统一为"可 git 管理工件"（`Application.java:21 implements Artifact`）

---

## 5. 前后端交互

### 5.1 API 层（src/api）

- 基类 `src/api/Api.ts:12-19`：axios 实例，`baseURL: "/api/"`，`withCredentials`，请求/响应拦截器（`src/api/interceptors`，处理 ResponseDTO 包装、401 等）
- 新式轻封装 `src/api/core/api.ts` + `factory.ts`（get/post/put/patch/delete 泛型）
- 域 API 类：`PageApi`（`static url = "v1/pages"`）、ActionAPI、GitSyncAPI、OAuthApi、PluginApi、AppThemingApi 等；`ce/api/` + `ee/api/` 空壳扩展

### 5.2 页面 DSL 的持久化接口

`PageApi.tsx:209-227`：

```
PUT v1/layouts/{layoutId}/pages/{pageId}?applicationId=...   body: { dsl: <DSLWidget> }
```

对应服务端 `LayoutControllerCE` 的 `@PutMapping("/{layoutId}/pages/{branchedPageId}")`（`controllers/ce/LayoutControllerCE.java:64`）；另有 `PUT v1/layouts/application/{appId}` 批量、`PUT v1/layouts/refactor` 重命名。完整保存链路见《02-页面保存》。

### 5.3 首屏合并加载（关键优化）⭐

- `api/services/ConsolidatedPageLoadApi/api.ts:7-25` 的 `getConsolidatedPageLoadDataView/Edit` → `GET /api/v1/consolidated-api/{view|edit}`
- 服务端 `ConsolidatedAPIController.java:51,83` 返回 `ConsolidatedAPIResponseCE_DTO`（`dtos/ce/ConsolidatedAPIResponseCE_DTO.java:31-85`）：userProfile、featureFlags、organizationConfig、pages、published/unpublishedActions、actionCollections、themes、**pageWithMigratedDsl**（服务端迁移后的 DSL）、customJSLibraries、plugins、datasources、pluginFormConfigs、mockDatasources
- view 模式还计算 **ETag + Cache-Control 支持 304**（Controller:108-135）
- Saga 侧入口 `sagas/InitSagas.ts:281-288`（EDIT→/edit，PUBLISHED→/view）

### 5.4 DSL 数据格式（存哪、长什么样）

- **存储**：MongoDB `newPage` 集合内 `unpublishedPage.layouts[0].dsl`（JSONObject）与 `publishedPage.layouts[0].publishedDsl` —— **页面 widget 树不是独立集合，而是 NewPage 文档内嵌**
- 前端类型：`DSLWidget extends WidgetProps { children?: DSLWidget[] }`（`src/WidgetProvider/types.ts:49-51`）；运行态 Redux 里被 `flattenDSL` 成 `Record<widgetId, FlattenedWidgetProps>`（`canvasWidgetsReducer.ts:172`）
- 真实样例（`appsmith-server/src/test/resources/test_assets/DSLMigration/PageDSLv83.json`，节选）：

```json
{
  "widgetName": "MainContainer", "type": "CANVAS_WIDGET", "widgetId": "0",
  "snapColumns": 64, "snapRows": 124, "version": 83,
  "dynamicBindingPathList": [], "dynamicTriggerPathList": [],
  "children": [
    { "widgetName": "Audio1", "type": "AUDIO_WIDGET",
      "widgetId": "44e9x2cf6q", "parentId": "0",
      "leftColumn": 30, "topRow": 43,
      "dynamicBindingPathList": [{ "key": "accentColor" }],
      "accentColor": "{{appsmith.theme.colors.primaryColor}}" }
  ]
}
```

要点：**树形结构，以 `widgetId/parentId/children` 关联；`leftColumn/topRow/rightColumn/bottomRow` 网格定位；`{{binding}}` 动态表达式 + `dynamicBindingPathList` 显式声明；顶层带 DSL `version` 供迁移**。

- 迁移：客户端包 `app/client/packages/dsl`（`migrateDSL`、`LATEST_DSL_VERSION`、编号迁移脚本，`src/index.ts:1-14`）；服务端在返回 page 前也做 `migrateDsl`（ConsolidatedAPIResponse 里的 `pageWithMigratedDsl` 字段即来源）

---

## 6. 编辑器运行时 vs 生产（view）运行时

| 维度 | 编辑器（EDIT） | 查看器（PUBLISHED） |
|---|---|---|
| 页面组件 | `src/pages/AppIDE/` | `src/pages/AppViewer/`（`AppViewerPageContainer.tsx:31-100` → `AppPage.tsx:24-73`） |
| 首屏请求 | `GET /consolidated-api/edit` | `GET /consolidated-api/view`（带 ETag/304） |
| DSL 来源 | `NewPage.unpublishedPage` + `Layout.dsl` | `publishedPage` + `publishedDsl`（publish 时由 `ApplicationPageServiceCEImpl.publish*` 把 unpublished 拷贝到 published，:1105-1319） |
| Action 来源 | `NewAction.unpublishedAction` / `Application.pages` | `publishedAction` / `Application.publishedPages` |
| renderMode | `RenderModes.CANVAS`（`selectors/editorSelectors.tsx:285-292` `getRenderMode`） | `RenderModes.PAGE`（枚举 `constants/WidgetConstants.tsx:40-45`，另有 `CANVAS_SELECTED`、`COMPONENT_PANE`） |
| UI 差异 | 可拖拽/属性面板/选中框（同一套 widget 组件按 renderMode 分支） | 纯渲染 |

共同点：

- **同一套 widget 组件与布局系统**：`layoutSystems/withLayoutSystemWidgetHOC.tsx:22-34` 按 `LayoutSystemTypes.FIXED|AUTO|ANVIL`（`types.ts:6-10`）× renderMode 取对应 widgetSystem/canvasSystem；anvil/fixedlayout/autolayout 各自有 `editor/` 与 `viewer/` 两套实现目录
- **同一个 eval worker**：`utils/workerInstances.ts:3-13` 创建 `evaluation.worker.ts` module worker（RPC：sync/async handler map，`evaluation.worker.ts:78-79`）；view 模式也运行同样的 `{{ }}` 绑定求值、onLoad action 执行（`layoutOnLoadActions` 由服务端 `onload` 包计算好存进 Layout）
- 编辑器内预览：`MainPane.tsx:16-25` `selectCombinedPreviewMode` 时直接渲染 `WidgetsEditor`；`EditorReduxState.isPreviewMode`（`ce/reducers/uiReducers/editorReducer.tsx:363`，`SET_PREVIEW_MODE`:266）
- 辅助进程 **RTS**（Node，`app/client/packages/rts`）：提供 `/ast`（AST 分析）、`/dsl`（格式化）、`/git`（git 操作）HTTP 服务（`src/ce/server.ts:29-32`），编辑器场景使用

---

## 7. 关键目录速查表

| 目录 | 职责 |
|---|---|
| `app/client/src/index.tsx` | 前端入口：预加载 chunks、建 worker、注册 widgets、挂载 Provider+AppRouter |
| `app/client/src/ce/` | 社区版实现层（AppRouter、pages、reducers、sagas、api、constants、selectors、workers） |
| `app/client/src/ee/` | 企业版挂载层（shim 转发或 `extends CE_X` 空壳），组合根统一 import ee |
| `app/client/src/pages/AppIDE/` | 新版编辑器页面（layouts/routers/components：Sidebar/LeftPane/MainPane/RightPane） |
| `app/client/src/pages/AppViewer/` | 生产查看器（view 模式）页面与导航 |
| `app/client/src/IDE/` | 产品无关的 IDE 基础 UI 组件与接口（Toolbar/BottomView/EditableName） |
| `app/client/src/PluginActionEditor/` | 查询/API 编辑器可复用框架 |
| `app/client/src/reducers/` | 共享 reducer 主体（entity/ui/evaluation/linting 四组） |
| `app/client/src/sagas/` | 共享 saga 主体（Evaluations、PageSave、WidgetOperation、GitSync 等 ~60 个） |
| `app/client/src/actions/` · `selectors/` | Redux action creators 与 reselect 选择器 |
| `app/client/src/api/` | axios 基类 + 域 API 类 + ConsolidatedPageLoadApi 服务 |
| `app/client/src/widgets/` | 全部 widget 实现（BaseWidget 派生，~70 种） |
| `app/client/src/WidgetProvider/` | widget 注册工厂、DSLWidget/WidgetProps 类型 |
| `app/client/src/layoutSystems/` | 三套布局系统（fixed/autolayout/anvil），各含 editor/viewer 实现 |
| `app/client/src/workers/Evaluation/` | eval web worker（JS 求值、JSObject、lint、replay） |
| `app/client/src/entities/` | 前端领域类型（Page/Application/Datasource/DataTree/GitSync…） |
| `app/client/src/git/` + `git-artifact-helpers/` | 前端 git 状态、sagas、UI 辅助 |
| `app/client/packages/dsl/` | DSL 迁移/展平/嵌套工具包（独立 npm workspace） |
| `app/client/packages/rts/` | Node 实时服务：AST/DSL/git 端点 |
| `app/client/packages/{design-system,icons,ast,utils}` | ADS 设计系统、图标、AST、共享工具（workspace 子包） |
| `app/server/appsmith-server` | Spring Boot 主服务（controllers/services/repositories/domains + 各域子包） |
| `app/server/appsmith-interfaces` | `com.appsmith.external`：插件契约 PluginExecutor、共享模型（Datasource/ActionDTO/BaseDomain） |
| `app/server/appsmith-plugins/*` | 26 个数据源插件执行器（postgres、restApi、openAi、googleSheets…） |
| `app/server/appsmith-git` | git 序列化/执行服务（供 git 集成） |
| `app/server/reactive-caching` | 响应式 Redis 缓存注解与分布式锁 |
| `deploy/docker` · `deploy/helm` 等 | docker-compose / K8s / 云平台部署编排 |
| `scripts/` | 构建/发布/健康检查 CI 脚本 |
| `app/util/` | WSL 检测脚本（容器启动用） |

---

## 8. 易混淆点备注

1. **"组合根永远指向 ee"**：即使 CE 版，`src/index.tsx`、`store.ts` 也 import `ee/AppRouter`、`ee/reducers`、`ee/sagas`；`ee/` 里多数文件只是 `export * from "ce/..."` 转发或空继承壳。**搜索实现时若落在 ee shim，需跳到 ce 同路径。**
2. **服务端 REST 的 `{branchedPageId}/{branchedApplicationId}`**：源自 `RefAwareDomain.baseId/refType/refName`，git 分支场景下每个分支有独立文档、通过 baseId 关联回主线（前端 URL 中的 `basePageId`）。
3. **published/unpublished 双轨**贯穿 Application/NewPage/NewAction/Datasource 全部资源；"发布"就是把 unpublished 拷贝为 published。
4. **Consolidated API 是首屏唯一大请求**（edit/view 各一），旧的分散接口（`v1/pages/{id}`、`v1/actions/view`…）仍存在并在 DTO 注释中一一对应。
5. **eval worker 是共享内核**：编辑器与查看器都靠它求值 `{{ }}`，`SET_EVALUATED_TREE` 因高频在 Sentry 中被忽略（`store.ts:14-18`）。

---

## 9. 四大机制导读（指向专题篇）

| 问题 | 一句话答案 | 详见 |
|---|---|---|
| 页面如何保存？ | 所有画布操作收敛到 `updateAndSaveLayout` → 500ms 防抖 → PUT DSL → 服务端校验后全量覆盖 `newPage.unpublishedPage.layouts[0].dsl`（无乐观锁，后写胜） | [02-页面保存](02-page-save.md) |
| 组件如何抽象？ | Widget = 继承 `BaseWidget` 的类（静态 getter 声明元配置）+ 纯 UI `component/`；注册表懒加载汇入 `WidgetFactory`，HOC 链组装渲染 | [03-组件抽象](03-widget-abstraction.md) |
| 属性如何配置？ | 每个_Widget_ 用 `getPropertyPaneContentConfig/StyleConfig` 声明面板（section→property→controlType），控件注册表分发渲染，修改走统一 batch action 原子写回 DSL | [04-属性配置](04-property-config.md) |
| 数据如何绑定？ | 全实体合成 DataTree，依赖图拓扑排序，Web Worker 内 eval `{{ }}`，deep-diff 增量回传 Redux，副作用（run/navigateTo）以描述符发回主线程执行 | [05-数据绑定](05-data-binding.md) |
