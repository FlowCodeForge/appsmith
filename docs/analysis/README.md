# Appsmith 源码分析文档

> 对象：`app/client`（React + Redux 前端）与 `app/server`（Java Spring Boot WebFlux 后端）
> 行号基于当前 release 分支，供快速跳转定位，后续代码演进可能有偏移。

## 阅读顺序

| # | 文档 | 回答的问题 | 一句话摘要 |
|---|---|---|---|
| 1 | [01-architecture-overview.md](01-architecture-overview.md) | 系统长什么样？ | 前后端分层、CE/EE 镜像机制、域模型（Application→NewPage→Layout→DSL）、编辑器/查看器双运行时 |
| 2 | [02-page-save.md](02-page-save.md) | 页面如何保存？ | 画布操作 → `updateAndSaveLayout` → 500ms 防抖 → PUT DSL → 服务端校验 → MongoDB `newPage` 集合（无乐观锁，后写胜） |
| 3 | [03-widget-abstraction.md](03-widget-abstraction.md) | 组件如何抽象？ | `BaseWidget` 静态 getter 元配置 + 纯 UI component 两层分离；注册表懒加载汇入 `WidgetFactory`，HOC 洋葱链组装渲染 |
| 4 | [04-property-config.md](04-property-config.md) | 属性如何配置？ | propertyPaneConfig 声明面板（section→property→controlType），控件注册表分发，统一 batch action 原子写回 DSL 并触发重求值 |
| 5 | [05-data-binding.md](05-data-binding.md) | 数据如何绑定？ | 全实体合成 DataTree（双树），依赖图拓扑排序，Web Worker 内 eval `{{ }}`，deep-diff 增量回传；副作用以描述符回主线程执行 |

## 核心链路速查（跨文档）

```
拖拽/改属性 ──▶ Redux action ──▶ saga 计算新 widgets ──▶ updateAndSaveLayout (UPDATE_LAYOUT)
                                   │                          │
                                   │                          ├─ canvasWidgetsReducer（UI 即时刷新）
                                   │                          └─ saveLayoutSaga → debounce(500) → savePageSaga
                                   │                                 → nestDSL → PUT v1/layouts/... → MongoDB newPage
                                   ▼
                            EvaluationsSaga（求值白名单命中）
                                   → getUnevaluatedDataTree（DataTree 双树组装）
                                   → evalWorker EVAL_TREE（依赖图 + 拓扑序 + eval {{}}）
                                   → SET_EVALUATED_TREE（deep-diff patch）
                                   → withWidgetProps 合成 props → React 重渲染
```

## 术语表

| 术语 | 含义 | 定义处 |
|---|---|---|
| DSL | 页面 widget 树的 JSON 描述（树形，widgetId/parentId/children 关联，网格定位，`version` 供迁移） | `app/client/src/WidgetProvider/types.ts:49` |
| Widget | 低代码组件；逻辑层（BaseWidget 子类）+ 表现层（component/） | `app/client/src/widgets/BaseWidget.tsx:79` |
| PropertyPane | 右侧属性面板，由 propertyPaneConfig 声明式描述 | `app/client/src/constants/PropertyControlConstants.tsx` |
| DataTree | widgets + actions + JS 对象 + `appsmith` 全局合成的求值上下文树 | `app/client/src/selectors/dataTreeSelectors.ts:146` |
| dynamicBindingPathList | widget 上需每轮求值的 `{{}}` 属性路径表 | `app/client/src/utils/DynamicBindingUtils.ts:176` |
| dynamicTriggerPathList | 事件触发型属性（onClick 等）路径表，惰性执行 | 同上 |
| eval worker | 专用 Web Worker，承载 DataTree 求值引擎 | `app/client/src/workers/Evaluation/evaluation.worker.ts` |
| CE / EE | 社区版 / 企业版；前端靠 `src/ce` ↔ `src/ee` 镜像目录 + ee 别名切换 | `app/client/src/enterprise/README.md` |
| published / unpublished | 草稿与发布双轨存储，贯穿 Application/NewPage/NewAction | `app/server/.../domains/NewPage.java` |
| branchedPageId | git 分支场景的页面 id（RefAwareDomain.baseId 关联主线） | `appsmith-interfaces/.../RefAwareDomain.java:13` |
| Consolidated API | 首屏合并加载接口 `/consolidated-api/{edit|view}`，view 带 ETag/304 | `app/server/.../controllers/ConsolidatedAPIController.java` |
