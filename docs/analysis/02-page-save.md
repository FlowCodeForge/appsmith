# Appsmith 源码分析（二）：页面如何保存 —— 从画布操作到 MongoDB

> 本文梳理编辑器里页面的完整保存链路：widget 增删改如何进入 Redux、两级节流的保存 saga、API 调用、后端校验与持久化、版本与并发模型，以及三条保存路径的差异。
> 所有结论均标注代码位置（`文件路径:行号`），行号基于当前 release 分支。

---

## 1. 总链路图

```
用户操作（拖入/拖动/缩放/删除 widget、改属性面板）
  → Redux action（WIDGET_ADD_CHILD / WIDGETS_MOVE / WIDGET_RESIZE / WIDGET_DELETE / BATCH_UPDATE_WIDGET_PROPERTY ...）
  → 各 widget saga 计算出新的「扁平化 widget 表」(canvasWidgets)
  → 统一汇聚：updateAndSaveLayout() → UPDATE_LAYOUT
      ├─ reducer 立即更新 canvasWidgets（UI 即时刷新）
      └─ saga: saveLayoutSaga → 权限/模式检查 → dispatch saveLayout() → SAVE_PAGE_INIT
          → redux-saga debounce(500ms) → savePageSaga
          → select(getWidgets) + nestDSL() 把扁平表还原为嵌套 DSL
          → PageApi.savePage → PUT /v1/layouts/{layoutId}/pages/{pageId}?applicationId=...
          → LayoutControllerCE.updateLayout
          → UpdateLayoutServiceCEImpl.updateLayout / updateLayoutDsl
              ├─ 校验 dynamicBindingPathList（非法引用直接抛错 400）
              ├─ 提取 widgetNames、mustache 绑定；转义 Mongo 特殊字符 key
              └─ 计算页 onLoad 动作 DAG（依赖图）
          → ExecutableOnPageLoadServiceCEImpl.findAndUpdateLayout
              ├─ applicationService.saveLastEditInformation（记录 lastEditedBy/At）
              └─ NewPageServiceCEImpl.saveUnpublishedPage → repository.save(newPage)
          → MongoDB 集合 "newPage"（unpublishedPage.layouts[0].dsl）
          ← LayoutDTO（含 actionUpdates/messages/layoutOnLoadActionErrors）
  → 前端 SAVE_PAGE_SUCCESS：更新 pageActions、setLastUpdatedTime、runBehaviour 更新
```

**核心设计：所有 widget 操作最终都收敛到一个 action —— `updateAndSaveLayout()`。**

---

## 2. 前端：widget 增删改如何进入 Redux

### 2.1 拖入新 widget（widget 库 → 画布）

- widget 库卡片按下即进入拖拽态：`app/client/src/pages/Editor/widgetSidebar/WidgetCard.tsx:113-138`，hook `setDraggingNewWidget` → `SET_NEW_WIDGET_DRAGGING`（hook 定义 `app/client/src/utils/hooks/dragResizeHooks.tsx:109-125`）
- 固定布局画布 drop 计算：`app/client/src/layoutSystems/fixedlayout/editor/FixedLayoutCanvasArenas/hooks/useBlocksToBeDraggedOnCanvas.ts`
  - 已有 widget 移动：`bulkMoveWidgets` → dispatch `WIDGETS_MOVE`（263-273 行）
  - 新 widget 落布：`addNewWidget` → dispatch `WIDGETS_ADD_CHILD_AND_MOVE` 或 `updateWidget(ADD_CHILD...)` 即 `WIDGET_ADD_CHILD`（275-297 行）
  - 本 hook 被 `useCanvasDragging.ts:91` 使用（Auto 版在 `layoutSystems/autolayout/editor/AutoLayoutCanvasArenas/hooks/`，同构）
- action 构造器：`updateWidget()` 在 `app/client/src/actions/pageActions.tsx:443-460`（`WidgetReduxActionTypes["WIDGET_" + operation]`）；动作类型表 `app/client/src/ce/constants/ReduxActionConstants.tsx:1459-1469`（WidgetReduxActionTypes）与 512-603（WidgetCanvasActionTypes）

### 2.2 拖动已有 widget

`WIDGETS_MOVE` → `moveWidgetsSaga`（watcher `app/client/src/sagas/CanvasSagas/DraggingCanvasSagas.ts:519-524`，saga 主体在同文件，碰撞/父子容器变更/reflow 计算后 417 行）：

```ts
yield put(updateAndSaveLayout(updatedWidgetsOnMove))
```

### 2.3 缩放

- UI 入口：`app/client/src/layoutSystems/common/resizer/ResizableComponent.tsx:229` `updateWidget(WidgetOperations.RESIZE, ...)`
- `WIDGET_RESIZE` → `resizeSaga`：`app/client/src/sagas/WidgetOperationSagas.tsx:173`，278 行 `put(updateAndSaveLayout(...))`（watcher 2066 行）

### 2.4 删除

- 入口 action：`deleteSelectedWidget`（`app/client/src/actions/widgetActions.tsx:116-127`）→ `WIDGET_DELETE`
- watcher：`app/client/src/sagas/WidgetDeletionSagas.ts:594-602`（`deleteSagaInit`:162 → `deleteSaga`:239），最终同样 `updateAndSaveLayout`
- 快捷键（backspace/del/mod+x/mod+c/mod+v 等）统一在 `app/client/src/pages/Editor/GlobalHotKeys/GlobalHotKeys.tsx:243-297` dispatch

### 2.5 属性面板改属性（最常用路径）

- 属性控件：`app/client/src/pages/Editor/PropertyPane/PropertyControl.tsx:269-277` `onBatchUpdateProperties` → `batchUpdateWidgetProperty(widgetId, { modify })`；action 定义 `app/client/src/actions/controlActions.tsx:30-41`（`BATCH_UPDATE_WIDGET_PROPERTY`）
- `BATCH_UPDATE_WIDGET_PROPERTY` 用 `actionChannel` **串行消费**（保证状态顺序）：`WidgetOperationSagas.tsx:1934-1952` `widgetBatchUpdatePropertySaga`
- 真正处理：`batchUpdateWidgetPropertySaga`（`WidgetOperationSagas.tsx:801-837`）→ `getPropertiesToUpdate`/`computeWidgetProperties`（635/751 行，**维护 dynamicBindingPathList / dynamicTriggerPathList / dynamicPropertyPathList**）→ 829 行 `put(updateAndSaveLayout(widgets, { shouldReplay }))`
- JS 模式切换：`setWidgetDynamicPropertySaga`（597 行）、`batchUpdateWidgetDynamicPropertySaga`（579 行）同样落到 `updateAndSaveLayout`

### 2.6 新版 Anvil 布局（WDS/section-zone）

- drop 入口：`app/client/src/layoutSystems/anvil/editor/canvasArenas/hooks/useAnvilWidgetDrop.ts:43,59` → `addNewAnvilWidgetAction`（`ANVIL_ADD_NEW_WIDGET`）/ `moveAnvilWidgets`（`ANVIL_MOVE_WIDGET`），action 定义 `app/client/src/layoutSystems/anvil/integrations/actions/draggingActions.ts:24-65`
- saga：`anvilWidgetAdditionSagas/index.ts:169 addWidgetsSaga`（watcher 262 行）、`anvilDraggingSagas/index.ts:29 moveWidgetsSaga`（watcher 145 行）
- 两者都调 `updateAndSaveAnvilLayout`（`app/client/src/layoutSystems/anvil/utils/anvilChecksUtils.ts:14`，清理空 section、重分配 zone 空间），最后 89 行仍 `put(updateAndSaveLayout(...))`

### 2.7 auto-layout（旧弹性布局）

`AUTOLAYOUT_ADD_NEW_WIDGETS` / `AUTOLAYOUT_REORDER_WIDGETS`：`app/client/src/sagas/CanvasSagas/AutoLayoutDraggingSagas.ts:106,160` → `updateAndSaveLayout`；watcher 315-322 行。

---

## 3. 保存触发机制（saga、防抖、批量）

- 汇聚 action：`updateAndSaveLayout` → `UPDATE_LAYOUT`：`app/client/src/actions/pageActions.tsx:183-193`
- **两级节流**（watcher 注册 `app/client/src/ee/sagas/PageSagas.tsx:37-85`）：

| 层 | 代码 | 作用 |
|---|---|---|
| 第一级 | `takeLatest(UPDATE_LAYOUT, saveLayoutSaga)`（44 行） | 只保留最新一次布局更新 |
| 第二级 | `debounce(500, SAVE_PAGE_INIT, savePageSaga)`（53 行） | 真正的网络保存做 500ms 防抖 |

### 3.1 saveLayoutSaga（权限 + 模式闸门）

`app/client/src/ce/sagas/PageSagas.tsx:675-711`：

- 686-698 行：权限检查 `getHasManagePagePermission`（GAC 功能开关下校验页面 userPermissions，无权限直接 403 风格 `validateResponse`）
- 700-702 行：**仅当 `appMode === EDIT && !isPreviewMode`** 才 `put(saveLayout(...))` → `SAVE_PAGE_INIT`（action 构造器 `pageActions.tsx:195-200`）

### 3.2 savePageSaga（保存主 saga）

`app/client/src/ce/sagas/PageSagas.tsx:507-653`：

1. 508 行 `select(getWidgets)` 取最新扁平 widget 表（**防抖后取的是最终态**）
2. 520 行 `getLayoutSavePayload` → **只做一件事：`nestDSL()` 扁平转嵌套**（`app/client/src/ce/sagas/helpers.ts:102-112`；nestDSL/flattenDSL 基于 normalizr，`app/client/packages/dsl/src/transform/lib.ts:16/28`）。前端保存时不再做版本号处理，DSL 根节点 `version` 字段随树原样保留
3. 528-540 行：先把最新 DSL 写进 pageDSLs reducer（`FETCH_PAGE_DSL_SUCCESS`）和 `UPDATE_CANVAS_STRUCTURE`
4. 552 行 `call(PageApi.savePage, savePageRequest)`
5. 556 行 `validateResponse`；成功后 590-591 行 `setLastUpdatedTime` + `savePageSuccess`
6. 失败分支 611-650 行：若 `IncorrectBindingError`（服务端校验出非法绑定路径），前端做一次**自愈**：`migrateIncorrectDynamicBindingPathLists`（`app/client/src/utils/migrations/IncorrectDynamicBindingPathLists.ts`）→ 重新 `updateAndSaveLayout(normalizedWidgets, { isRetry: true })`

### 3.3 保存状态 UI

- reducer：`app/client/src/ce/reducers/uiReducers/editorReducer.tsx:145-167`（`SAVE_PAGE_INIT → saving:true`，`SAVE_PAGE_SUCCESS/ERROR`）
- 展示组件：`app/client/src/pages/Editor/EditorSaveIndicator.tsx`（仅"保存中/保存失败"两种显示），AppIDE 头部挂载 `app/client/src/pages/AppIDE/layouts/components/Header/index.tsx:124,160`

---

## 4. API 调用

`app/client/src/api/PageApi.tsx`：

| 方法 | 位置 | 说明 |
|---|---|---|
| `savePage` | 209-227 行 | `PUT v1/layouts/{layoutId}/pages/{pageId}?applicationId={applicationId}`，body `{ dsl }`。**并发控制：发起前 `pageUpdateCancelTokenSource.cancel()` 取消上一个未完成的保存请求**（axios CancelToken，191/212-218 行） |
| `saveAllPages` | 229-236 行 | `PUT v1/layouts/application/{applicationId}`，body `{ pageLayouts }` —— 用于布局系统转换（`app/client/src/sagas/layoutConversionSagas.ts:85,173` 调 `saveAllPagesSaga`） |
| `fetchPage`（对照） | 199-207 行 | `GET v1/pages/{pageId}?migrateDsl=true` |

---

## 5. 后端：controller → service → 持久化

### 5.1 Controller

`app/server/appsmith-server/src/main/java/com/appsmith/server/controllers/ce/LayoutControllerCE.java`：

- 63-74 行：`@PutMapping("/{layoutId}/pages/{branchedPageId}")` `updateLayout(...)` → `updateLayoutService.updateLayout(branchedPageId, applicationId, layoutId, dto.toLayout())`；`LayoutUpdateDTO` 是 record，只含 `dsl`（`server/dtos/LayoutUpdateDTO.java:6-10`）
- 53-61 行：`PUT /application/{applicationId}` `updateMultipleLayouts`（批量保存所有页面）
- URL 常量：`server/constants/ce/UrlCE.java:12` `LAYOUT_URL = /v1/layouts`

### 5.2 Service（保存时的校验/处理）

`app/server/appsmith-server/src/main/java/com/appsmith/server/layouts/UpdateLayoutServiceCEImpl.java`：

- `updateLayout`（292-311 行）：按 application 的 evaluationVersion 分支
- `updateLayoutDsl`（120-284 行）核心流程：
  1. **`extractAllWidgetNamesAndDynamicBindingsFromDSL`**（516-695 行）：遍历 DSL 收集 widgetNames；**逐个验证 `dynamicBindingPathList` 里的路径存在且确有 mustache 绑定，否则抛 `AppsmithError.INVALID_DYNAMIC_BINDING_REFERENCE`**（577-643 行，即前端 `IncorrectBindingError` 的来源）
  2. `removeSpecialCharactersFromKeys`（697-706 行）：Table widget 主列名含 `$`/`.` 等 Mongo 特殊字符时转义（`escapeTableWidgetPrimaryColumns`），记录到 `mongoEscapedWidgetNames`，返回响应前再 `unescapeMongoSpecialCharacters`（342-355 行）
  3. `onLoadExecutablesUtil.findAllOnLoadExecutables`（204-226 行）：解析 mustache 引用构建依赖 DAG，**计算页加载时需执行的 action/JS**；循环依赖时不更新 action，写 `layoutOnLoadActionErrors`（前端 toast 提示 + debugger）
  4. `updateExecutablesRunBehaviour`（243-253 行）：把 onLoad 计算结果写回 action 的 runBehaviour，变更通过响应的 `actionUpdates`/`messages` 返回，前端在 `ce/PageSagas.tsx:572-587` dispatch `setActionsRunBehaviour`/`setJSActionsRunBehaviour`
  5. 263-268 行调 `onLoadExecutablesUtil.findAndUpdateLayout(...)` 持久化

### 5.3 持久化

`app/server/appsmith-server/src/main/java/com/appsmith/server/newpages/onload/ExecutableOnPageLoadServiceCEImpl.java:78-120` `findAndUpdateLayout`：

- `newPageService.findByIdAndLayoutsId(creatorId, layoutId, EDIT 权限, false)` —— **权限即 ACL**（无权限返回 `ACL_NO_RESOURCE_FOUND`，404）
- 用 `BeanUtils.copyProperties(layout, storedLayout)` 整体替换目标 layout（98 行）
- 105-108 行：先 `applicationService.saveLastEditInformation(applicationId)`（`applications/base/ApplicationServiceCEImpl.java:823-842`：set `lastEditedAt`、`isManualUpdate=true`、`modifiedBy=当前用户`，仅 `$set` 更新 application）
- 109 行：`newPageService.saveUnpublishedPage(page)`

`app/server/appsmith-server/src/main/java/com/appsmith/server/newpages/base/NewPageServiceCEImpl.java:157-169` `saveUnpublishedPage`：

- `findById` → `newPage.setUnpublishedPage(page)`（**只写未发布版**）→ 首次补 `gitSyncId`（`applicationId + UUID`）→ `repository.save(newPage)`

### 5.4 存储

- 领域对象 `server/domains/NewPage.java:19-21`：`@Document`（无自定义 collection 名）→ **MongoDB 集合 `newPage`**（迁移脚本 `migrations/db/ce/Migration059PolicySetToPolicyMap.java:57` 等直接以 `"newPage"` 引用该集合）
- DSL 存在 `NewPage.unpublishedPage.layouts[0].dsl`（net.minidev `JSONObject`）
- `Layout.java:33-133` 字段：`dsl`（未发布）/ `publishedDsl`（发布版，44-45 行）、`layoutOnLoadActions`/`publishedLayoutOnLoadActions`、`widgetNames`、`mongoEscapedWidgetNames`、`layoutOnLoadActionErrors`、`id`

### 5.5 DSL 版本迁移发生在"读取"时（不是保存时）

**服务端保存时不做 DSL 版本迁移**。迁移发生在「读取」时：

- 服务端：`services/ce/ApplicationPageServiceCEImpl.java:341-347` `getPageAndMigrateDslByBranchedPageId` → `migrateAndUpdatePageDsl`（349-390 行）→ `dslMigrationUtils.getLatestDslVersion()/migratePageDsl()`（调 RTS 服务）并把迁移后的 DSL 回写 DB
- 客户端：`utils/WidgetPropsUtils.tsx:47-77` `extractCurrentDSL` → `migrateDSL`（`app/client/packages/dsl/src/migrate/index.ts:657`，`LATEST_DSL_VERSION = 94`，共 92 个版本迁移脚本，同目录 `migrations/`）
- 因此若 fetch 时发生迁移或 Anvil 转换，前端会立即回存：`ce/PageSagas.tsx:305-306` `if (willPageBeMigrated || isAnvilLayout) yield put(saveLayout())`

---

## 6. 版本与并发

| 问题 | 结论 |
|---|---|
| 页面/DSL 乐观锁 | **没有**：`BaseDomain`（`appsmith-interfaces/src/main/java/com/appsmith/external/models/BaseDomain.java:40+`）只有 `@Id/createdAt/updatedAt/createdBy/lastModifiedBy`，无 `@Version`；`saveUnpublishedPage` 是 read-modify-write 全量覆盖 → **last-write-wins（后保存者胜）** |
| 并发防护 | 主要在前端：① `takeLatest(UPDATE_LAYOUT)` + `debounce(500, SAVE_PAGE_INIT)` 合并抖动；② `PageApi.savePage` 取消上一个在途请求（CancelToken，`PageApi.tsx:212-218`）；③ BATCH_UPDATE_WIDGET_PROPERTY 的 actionChannel 串行 |
| 多人编辑提示 | `saveLastEditInformation` 更新 application 的 `lastEditedAt/modifiedBy`（供"他人已编辑"类 UI 使用）；另有 `REFRESH_THE_APP` action（常量 `ce/constants/ReduxActionConstants.tsx:594`，saga `refreshTheApp`，`ce/PageSagas.tsx:183`） |
| git 分支模式 | 路由/接口全部用 `branchedPageId`；git 连接的应用每个分支各有一套 NewPage 文档（`NewPage.baseId`、`gitSyncId` 跨实例同步标识，`NewPageServiceCEImpl.java:162-165`）。提交/合并由 GitSyncSagas + 服务端 git 序列化处理，不改变单页保存链路 |
| DSL 版本号 | 是 DSL JSON 根节点的 `version` 字段（迁移时逐级 +1 至 94），**不是数据库列** |

---

## 7. 三条保存路径

| 路径 | 实际行为 |
|---|---|
| **自动保存（唯一真正保存路径）** | 任意 `UPDATE_LAYOUT` → `saveLayoutSaga` → `SAVE_PAGE_INIT` → `debounce(500)` → `savePageSaga` → `PageApi.savePage`（见第 3、4 节） |
| **Cmd+S / Ctrl+S** | 被 `GlobalHotKeys.tsx:380-391`（`combo: "mod + s"`）拦截，仅 `toast.show(SAVE_HOTKEY_TOASTER_MESSAGE)`（文案 "Don't worry about saving, we've got you covered!"，`ce/constants/messages.ts:317-318`），**preventDefault + 不触发任何保存** |
| **"手动保存"类场景** | 本版本无保存按钮，顶栏只有保存状态指示器（`EditorSaveIndicator.tsx`）。相关兜底：① fetch 时若 DSL 被迁移/Anvil 转换则自动 `saveLayout()` 回存（`ce/PageSagas.tsx:305`）；② 模板流程 `TemplatesSagas.ts:374`、Building Block 添加 `BuildingBlockSagas/BuildingBlockAdditionSagas.ts:199` 等会 `take(SAVE_PAGE_SUCCESS)` 等待保存完成；③ 布局系统切换用批量接口 `saveAllPagesSaga`（`layoutConversionSagas.ts:85,173`） |

---

## 8. 关键函数速查表

| 函数 | 位置 |
|---|---|
| `updateAndSaveLayout`（UPDATE_LAYOUT action 构造） | app/client/src/actions/pageActions.tsx:183 |
| `saveLayout`（SAVE_PAGE_INIT action 构造） | app/client/src/actions/pageActions.tsx:195 |
| `saveLayoutSaga`（权限+模式闸门） | app/client/src/ce/sagas/PageSagas.tsx:675 |
| `savePageSaga`（保存主 saga） | app/client/src/ce/sagas/PageSagas.tsx:507 |
| `getLayoutSavePayload`（nestDSL） | app/client/src/ce/sagas/helpers.ts:102 |
| watcher（takeLatest/debounce 500） | app/client/src/ee/sagas/PageSagas.tsx:44,53 |
| `PageApi.savePage`（PUT + cancel token） | app/client/src/api/PageApi.tsx:209 |
| `batchUpdateWidgetPropertySaga` / `getPropertiesToUpdate` | app/client/src/sagas/WidgetOperationSagas.tsx:801 / 635 |
| `addChildSaga` / `deleteSagaInit` / `moveWidgetsSaga` / `resizeSaga` | WidgetAdditionSagas.ts:400 / WidgetDeletionSagas.ts:162 / DraggingCanvasSagas.ts:519 / WidgetOperationSagas.tsx:173 |
| `updateAndSaveAnvilLayout` | app/client/src/layoutSystems/anvil/utils/anvilChecksUtils.ts:14 |
| `flattenDSL` / `nestDSL`（normalizr） | app/client/packages/dsl/src/transform/lib.ts:16 / 28 |
| `migrateDSL`（LATEST_DSL_VERSION=94） | app/client/packages/dsl/src/migrate/index.ts:657 / 98 |
| `extractCurrentDSL`（fetch 时迁移） | app/client/src/utils/WidgetPropsUtils.tsx:47 |
| `LayoutControllerCE.updateLayout`（PUT 端点） | app/server/.../controllers/ce/LayoutControllerCE.java:63 |
| `UpdateLayoutServiceCEImpl.updateLayout / updateLayoutDsl` | app/server/.../layouts/UpdateLayoutServiceCEImpl.java:292 / 120 |
| `extractAllWidgetNamesAndDynamicBindingsFromDSL`（绑定校验） | 同上:516 |
| `ExecutableOnPageLoadServiceCEImpl.findAndUpdateLayout` | app/server/.../newpages/onload/ExecutableOnPageLoadServiceCEImpl.java:78 |
| `NewPageServiceCEImpl.saveUnpublishedPage`（最终落库） | app/server/.../newpages/base/NewPageServiceCEImpl.java:157 |
| `ApplicationServiceCEImpl.saveLastEditInformation` | app/server/.../applications/base/ApplicationServiceCEImpl.java:823 |
| `getPageAndMigrateDslByBranchedPageId` / `migrateAndUpdatePageDsl`（读时迁移） | app/server/.../services/ce/ApplicationPageServiceCEImpl.java:341 / 349 |

saga 总注册在 `app/client/src/ce/sagas/index.tsx:67（pageSagas）,72（widgetOperationSagas）,98（draggingCanvasSagas）,109（autoLayoutDraggingSagas）,115（anvilSagas）`。

读取链路对照：`GET v1/pages/{id}?migrateDsl=true` → `getCanvasWidgetsPayload`（`ce/PageSagas.tsx:211`，extractCurrentDSL + flattenDSL）→ `initCanvasLayout` → canvasWidgets reducer。

---

## 9. 设计要点小结（可借鉴的抽象）

1. **单一汇聚点**：所有画布操作（增删改移缩、Anvil/AutoLayout）都收敛到 `updateAndSaveLayout` 一个 action，保存逻辑只有一份。
2. **两级节流 + 请求取消**：takeLatest 合并高频操作、500ms debounce 聚焦停顿、axios CancelToken 取消在途请求 —— 三层防抖动/防乱序。
3. **扁平/嵌套双形态**：Redux 内扁平 map（O(1) 更新），持久化时 nestDSL 还原树 —— 兼顾编辑性能与存储可读性。
4. **发布/未发布双版本**：`NewPage.unpublishedPage` 与 `publishedPage` 分离，保存只写未发布版 —— 发布即快照。
5. **读时迁移**：DSL 版本迁移放在读取路径（服务端 RTS + 客户端 92 个迁移脚本），保存路径保持简单。
6. **服务端兜底校验**：dynamicBindingPathList 合法性、Mongo 特殊字符转义、onLoad DAG 计算都在后端做 —— 前端 bug 不会写坏数据。
