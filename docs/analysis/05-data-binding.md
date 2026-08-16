# Appsmith 源码分析（五）：数据如何绑定 —— DataTree × `{{ }}` 求值引擎

> 本文梳理 `{{ }}` 表达式体系：DataTree 双树结构、Web Worker 求值引擎、动态路径表、三类数据源（API/Query、Datasource、JS Object）、事件触发执行链路、依赖图与增量重算。
> 所有结论均标注代码位置（`文件路径:行号`），行号基于当前 release 分支。

---

## 1. 总览：三棵树 + 一个 Worker

业务上一句话：**所有实体（widget、API/Query、JS 对象、全局 `appsmith`）被合并成一棵 `DataTree`；需要求值的 `{{ }}` 属性由依赖图排出拓扑序，在专用 Web Worker 里 eval；求值结果以 deep-diff 增量回传主线程 patch 进 Redux，widget 由此拿到真实 props。**

| 树 | 内容 | 位置 |
|---|---|---|
| **UnEvalTree** | 原始未求值树（widget 属性仍是 `"{{Input1.text}}"` 字符串） | 主线程 reselect selector 派生 |
| **ConfigTree** | 元数据树（dynamicBindingPathList、bindingPaths、reactivePaths、triggerPaths、validationPaths、setter…） | 同上 |
| **evalTree** | 已求值树 + 依赖图（`DataTreeEvaluator` 单例维护） | worker 内 |

关键目录：

| 目录 | 职责 |
|---|---|
| `app/client/src/entities/DataTree/` | 树工厂（主线程） |
| `app/client/src/workers/Evaluation/` | worker 入口、handlers、fns、JSObject 解析 |
| `app/client/src/workers/common/DataTreeEvaluator/` | 求值器核心类 |
| `app/client/src/workers/common/DependencyMap/` | 依赖图 |
| `app/client/src/workers/Tern/` | Tern 补全 worker（独立进程） |
| `app/client/src/sagas/EvaluationsSaga.ts` | 主线程编排 |

> 注意：目录是 `src/workers/Evaluation/`（大写 E），不存在小写 `evaluation/`。

```
Redux (widgets/actions/jsObjects/meta)
   │ selector: getUnevaluatedDataTree（主线程组装 UnEvalTree + ConfigTree）
   ▼
evalWorker.request(EVAL_TREE, { unevalTree, widgetTypeConfigMap, theme, ... })
   │ worker: DataTreeEvaluator.setupFirstTree / setupUpdateTree（diff → 改依赖图 → 拓扑排序）
   │ evaluateTree：按 evalOrder 对每条 dynamicBindingPath 执行 evaluateSync（indirect eval）
   ▼
deep-diff 增量 updates ──postMessage──▶ 主线程 setEvaluatedTree
   │ treeReducer: applyChange 逐条 patch state.evaluations.tree
   ▼
withWidgetProps: useSelector(getWidgetEvalValues) → state.evaluations.tree[widgetName]
   │ createCanvasWidget(DSL 未求值值, evaluatedWidget) 合成最终 props
   ▼
React 重渲染
```

---

## 2. 生命周期一：一次数据绑定（`{{Input1.text}}` → 组件更新）

### 步骤 1：用户写绑定 → 动态路径表登记

1. 用户把 `Text1.text` 从 `abc` 改为 `{{Input1.text}}`，属性面板触发 widget 属性更新 → `batchUpdateWidgetPropertySaga`（`sagas/WidgetOperationSagas.tsx:801-837`，`getPropertiesUpdatedWidget` 计算新 widget，**包括维护 dynamicBindingPathList**）。
2. JS 模式切换时：`handleUpdateWidgetDynamicProperty`（`WidgetOperationSagas.tsx:511-560`）把路径 push 进 `dynamicPropertyPathList`。
3. saga `put(updateAndSaveLayout(widgets))` → `UPDATE_LAYOUT`。
4. `UPDATE_LAYOUT` / `UPDATE_WIDGET_PROPERTY` 都在求值触发白名单：`ce/actions/evaluationActionsList.ts:29-30`（`EVALUATE_REDUX_ACTIONS`），判定函数 `shouldTriggerEvaluation`（`actions/evaluationActions.ts:18-22`）。

### 步骤 2：主线程构建 UnEvalTree / ConfigTree

5. 求值循环 `evaluationChangeListenerSaga`（`sagas/EvaluationsSaga.ts:934-983`）用 `actionChannel(..., evalQueueBuffer())` 缓冲/去抖；循环体 `evaluationLoopWithDebounce`（`EvaluationsSaga.ts:985-1061`）。
6. `evalAndLintingHandler`（`EvaluationsSaga.ts:805-889`）→ `getUnevalTreeWithWidgetsRegistered`（890-930 行，**按需懒加载 widget 组件并注册进 WidgetFactory**）→ `select(getUnevaluatedDataTree)`。
7. **DataTree 组装点**：`selectors/dataTreeSelectors.ts:146-200` `getUnevaluatedDataTree`：
   - `DataTreeFactory.actions(...)`（98-104，API/Query 动作）
   - `DataTreeFactory.jsActions(...)`（106-109，JS 对象）
   - `DataTreeFactory.widgets(...)`（111-128，所有 widget，含 module inputs/instances）
   - `DataTreeFactory.metaWidgets(...)`（129-135）
   - 再补 `appsmith` 全局实体（store/theme/ui/window 尺寸/当前页名等，182-194 行）
8. 工厂实现：`entities/DataTree/dataTreeFactory.ts:24-157`，每个实体生成 `{ unEvalEntity, configEntity }`：
   - **widget 侧**：`dataTreeWidget.ts:377-432` `generateDataTreeWidget` —— 把 meta 属性（运行时用户输入值，如 Input.text）merge 进 widget 实体（403-412）；derived properties（`this.x` 改写为 `WidgetName.x`）加入 dynamicBindingPathList（226-238）；configEntity 携带 `bindingPaths/reactivePaths/triggerPaths/validationPaths/dependencyMap`（277-284、341-364，来源 `getAllPathsFromPropertyConfig`：`entities/Widget/utils.ts:253-366`，**从属性面板配置推导哪些字段可绑定/可触发/替换策略**）
   - **action 侧**：`ce/entities/DataTree/dataTreeAction.ts:14-116` —— `dynamicTriggerPathList = [{key:"run"},{key:"clear"}]`（81 行）；`data` 在未求值树中恒为 undefined（90 行，数据由 `updateActionData` 直写 evalTree）

### 步骤 3：发给 worker

9. `evaluateTreeSaga`（`EvaluationsSaga.ts:353-426`）打包 `EvalTreeRequestData`（385-407）→ `evalWorker.request(EVAL_WORKER_ACTIONS.EVAL_TREE, ...)`（409-414）。
10. worker 实例：`utils/workerInstances.ts:3-13`（`new Worker(new URL("../workers/Evaluation/evaluation.worker.ts"))`）；worker 侧消息分发 `workers/Evaluation/evaluation.worker.ts:18-79`，handler 注册表 `handlers/index.ts:27-70`。

### 步骤 4：worker 内求值

11. `evalTree` handler（`handlers/evalTree.ts:48-372`）：
    - **首次**（无 `dataTreeEvaluator`）：101-140 → `new DataTreeEvaluator` → `setupFirstTree`（`workers/common/DataTreeEvaluator/index.ts:258-399`：`getAllPaths` 全路径收集 → `createDependencyMap` 建依赖图 → `sortDependencies` 拓扑排序）→ `evalAndValidateFirstTree`
    - **增量**：194-255 → `setupUpdateTree`（`DataTreeEvaluator/index.ts:666-818`：deep-diff 新旧 unEvalTree → `updateDependencyMap` 增量改依赖图 → `setupTree` 计算本轮求值顺序）→ `evalAndValidateSubTree`
12. 核心循环 `evaluateTree`（`DataTreeEvaluator/index.ts:1114-1400+`），对 evalOrder 中每个全路径：
    - `isAPathDynamicBindingPath`（dynamicBindingPathList 命中）：1206-1210
    - `isPathDynamicTrigger`（触发型字段**跳过数据求值**）：1212
    - `requiresEval` 判定：1248-1251
    - 取 `evaluationSubstitutionType`（configTree.reactivePaths，默认 TEMPLATE）：1258-1260
    - `this.getDynamicValue(...)`：1281-1290
13. `getDynamicValue`（`DataTreeEvaluator/index.ts:1650-1769`）：
    - `getDynamicBindings` 把 `"Hello {{Input1.text}}"` 拆成 stringSegments + jsSnippets（`utils/DynamicBindingUtils.ts:90-120`；`getDynamicStringSegments`（40-87）是**手写括号配对解析**；`isDynamicValue` 用 `DATA_BIND_REGEX`）
    - 每个 snippet → `evaluateDynamicBoundValue` → `evaluateSync`
14. `evaluateSync`（`workers/Evaluation/evaluate.ts:369-464`）：
    - `getUserScriptToEvaluate`（317-337）：`sanitizeScript` → 选模板 `EvaluationScripts`（48-86，EXPRESSION 型即 `function $$closedFn(){ const $$result = <<string>>; return $$result } $$closedFn.call(THIS_CONTEXT)`）
    - 构建求值上下文 `createEvaluationContext`（255-288）：ARGUMENTS、THIS_CONTEXT、globalContext、`getDataTreeContext`（把每个实体的求值版挂到全局，见 `ce/workers/Evaluation/Actions.ts:43-115`；数据字段求值时 `removeEntityFunctions` **防止在数据字段里调 `Api1.run()`**）
    - **执行**：`indirectEval(script)`（`workers/Evaluation/indirectEval.ts:1-5`）—— `(1, eval)(script)` 间接 eval，只在全局作用域跑、拿不到闭包变量
    - Promise 结果直接抛 `FoundPromiseInSyncEvalError`（431-437，**数据字段不许 async**）
15. 多段/模板替换：`substituteDynamicBindingWithValues`（`workers/Evaluation/evaluationSubstitution.ts:135-161`）：TEMPLATE（字符串拼接）/ SMART_SUBSTITUTE（对象自动加引号）/ PARAMETER（SQL 预编译 `$1,$2`）。单段非 PARAMETER 直接返回原值不求值替换（1731-1738）。
16. 求值后处理（`evaluateTree` 的 switch，`DataTreeEvaluator/index.ts:1328-1390+`）：widget 走 `validateAndParseWidgetProperty`（校验+解析，如字符串转数字）→ 写回 `contextTree`（**供后续表达式引用已求值值**，1144-1150 注释）、`safeTree`（干净副本）、记录 evalMetaUpdates（值落到 widgetsMeta）。错误写入 `EntityName.__evaluation__.errors.<path>`（`DynamicBindingUtils.ts:367-443`）。

### 步骤 5：增量回传 + Redux patch

17. worker 生成 diff 更新：`generateOptimisedUpdatesAndSetPrevState`（`workers/Evaluation/helpers.ts`；新树整棵 `kind:"newTree"`，否则按 evaluationOrder 限定 diff 范围）。
18. 主线程 `updateDataTreeHandler`（`EvaluationsSaga.ts:219-341`）：`setEvaluatedTree(parsedUpdates)`（262 行）→ reducer `reducers/evaluationReducers/treeReducer.ts:13-44` 用 deep-diff `applyChange` 逐条 patch `state.evaluations.tree`；同时 `updateMetaState`（267-269）、`setDependencyMap`（319）、`updateTernDefinitions`（刷新补全，309-317）、可选 `EXECUTE_REACTIVE_QUERIES`（330-338）。

### 步骤 6：组件拿到新 props

19. 每个渲染的 widget 被 `withWidgetProps` 包裹：`widgets/withWidgetProps.tsx:89-91` `useSelector(getWidgetEvalValues)`（= `state.evaluations.tree[widgetName]`，`selectors/dataTreeSelectors.ts:228-231`）；203-206 行用 `createCanvasWidget(widget /*DSL 未求值*/, evaluatedWidget, config)` 合成最终 props → React 重渲染。

### 快捷路径：运行时用户输入（Input1 打字 → Text1 更新）

不走属性面板，而是：`widgets/InputWidget` 调 `updateWidgetMetaProperty`（`widgets/MetaHOC.tsx:146-156`，200ms 去抖）→ `syncUpdateWidgetMetaProperty` 更新 `widgetsMeta` reducer → `triggerEvalOnMetaUpdate` 发 `META_UPDATE_DEBOUNCED_EVAL`（在求值白名单 `evaluationActionsList.ts:33`）→ 走同一 `evaluateTreeSaga`；由于 meta 已并入 UnEvalTree（`dataTreeWidget.ts:403-414`），`setupUpdateTree` diff 出 `Input1.text` 变化 → 依赖图反向查到 `Text1.text` 需要重算。

---

## 3. 生命周期二：一次查询执行（按钮 onClick 执行 Query1 → 后端 → 回填 UI）

### 阶段 A：点击 → 触发型绑定求值（worker）

1. Button 点击：`widgets/ButtonWidget/widget/index.tsx:586-602` `onButtonClick` → `super.executeAction({ triggerPropertyName:"onClick", dynamicString: this.props.onClick, ... })`（onClick 的值是原始字符串 `{{Query1.run()}}`）。
2. `BaseWidget.executeAction`（`widgets/BaseWidget.tsx:206-227`）→ `this.context.executeAction`（EditorContext）。
3. Context 注入：`components/editorComponents/EditorContextProvider.tsx:217-256` → dispatch `EXECUTE_TRIGGER_REQUEST`（`actions/widgetActions.tsx:19-25`，batchAction 包裹）。
4. 监听：`ce/sagas/ActionExecution/ActionExecutionSagas.ts:206-221` takeEvery → `initiateActionTriggerExecution`（181-204，先清旧错误）→ `executeAppAction`（146-179）。
5. `evaluateAndExecuteDynamicTrigger`（`sagas/EvaluationsSaga.ts:452-498`）：重新取最新树 → `evalWorker.request(EVAL_TRIGGER, ...)`。
6. worker 侧 handler `handlers/evalTrigger.ts:6-51`：
   - 先 `setupUpdateTree` + `evaluateAndPushResponse` 把未同步的树变化补算并推给主线程（23-38）
   - `dataTreeEvaluator.evaluateTriggers(...)` → `evaluateAsync`（`evaluate.ts:466-516`）：TRIGGERS 模板 `async function $$closedFn(){ const $$result = <<string>>; return await $$result }`，全局上下文此时**带实体函数**（`Actions.ts:62-96`，run/clear/setters 仅在 isTriggerBased 时挂上）
7. 脚本执行 `Query1.run()` → 实体函数 `run`（`workers/Evaluation/fns/actionFns.ts:36-102`）：`promisify(runFnDescriptor)`（描述符 `{type:"RUN_PLUGIN_ACTION", payload:{actionId, params, onSuccess, onError}}`）→ `WorkerMessenger.request(MAIN_THREAD_ACTION.PROCESS_TRIGGER)` **把 trigger 发回主线程并 await Promise**（`fns/utils/Promisify.ts:11-40`）。

### 阶段 B：主线程执行动作 → 后端

8. 主线程 `handleEvalWorkerRequestSaga`（`sagas/EvalWorkerActionSagas.ts:32-40`）→ case `PROCESS_TRIGGER`（139-141）→ `processTriggerHandler`（102-118）→ `executeTriggerRequestSaga`（`EvaluationsSaga.ts:512-547`，结果/异常回填给 worker 的 pending promise）。
9. 触发路由总开关：`ce/sagas/ActionExecution/ActionExecutionSagas.ts:63-144` —— switch trigger.type：RUN_PLUGIN_ACTION / NAVIGATE_TO / SHOW_ALERT / SHOW_MODAL_BY_NAME / STORE_VALUE 等（全部触发函数注册表：`workers/Evaluation/fns/index.ts:73-180, 202-224`）。
10. `RUN_PLUGIN_ACTION` → `executePluginActionTriggerSaga`（`sagas/ActionExecution/PluginActionSaga.ts:543-682`）：select 取 action/datasource/plugin、confirm 弹窗 → `executePluginActionSaga`（596-603）。
11. `executePluginActionSaga`（`PluginActionSaga.ts:1352-1576`）：
    - **先求值查询参数绑定**：`evaluateActionParams(pluginAction.jsonPathKeys, ...)`（1418-1425，内部走 `EVAL_ACTION_BINDINGS` worker 请求 → `handlers/evalActionBindings.ts`）
    - 组 FormData（含 blob 引用防大数据序列化）
    - **HTTP**：`ActionAPI.executeAction`（1441 行）→ `api/ActionAPI.tsx:277-284`（POST `v1/actions/execute`，multipart，带超时/取消 token）
12. 后端：`app/server/appsmith-server/src/main/java/com/appsmith/server/controllers/ce/ActionControllerCE.java:77-88`（`@PostMapping("/execute")`）→ `ActionExecutionSolutionCEImpl.executeAction`（`solutions/ce/ActionExecutionSolutionCEImpl.java:302-410`：解析 multipart → `getValidActionForExecution` 校验动作/权限 → 取 `DatasourceStorage`、`Plugin`、`PluginExecutor`（插件执行器来自独立 plugin 包/插件进程）→ `getActionExecutionResult` 真正执行 → `addDataTypesAndSetSuggestedWidget` 给结果标数据类型）。

### 阶段 C：结果回填

13. 响应处理（`PluginActionSaga.ts:1443-1468`）：`executePluginActionSuccess`（写入 actionsReducer 的 `data/isLoading/responseMeta`）→ **`put(updateActionData([{ entityName: "Query1", dataPath: "data", data: payload.body }]))`**（失败分支 1528-1539 用 EMPTY_RESPONSE）。返回 `[body, params, {isExecutionSuccess, statusCode, headers}]` 给 worker 的 await。
14. `UPDATE_ACTION_DATA` 进入求值通道（`EvaluationsSaga.ts:995-1002`）→ worker `UPDATE_ACTION_DATA` → `handlers/updateActionData.ts:10-66`：
    - `updateActionsToEvalTree`（42-66）：`set(evalTree, "Query1.[data]", data)` + `set(self, ...)`（更新 worker 全局上下文）+ DataStore
    - `evalTreeWithChanges({ updatedValuePaths: [["Query1","data"]] })`（32-39）
15. `evalTreeWithChanges`（`workers/Evaluation/evalTreeWithChanges.ts:43-87`）：`setupUpdateTreeWithDifferences`（`DataTreeEvaluator/index.ts:932-955`——不打 diff，直接给定变更路径）→ `evaluateAndPushResponse` → 重算所有依赖 `Query1.data` 的绑定 → 增量 → `pushResponseToMainThread`。
16. 主线程 `EvalWorkerActionSagas.ts:166-181` case `UPDATE_DATATREE` → `updateDataTreeHandler` → `SET_EVALUATED_TREE` patch → `withWidgetProps` 重渲染（例如 Table 的 `tableData: {{Query1.data}}` 拿到新数组）。
17. 页面加载自动执行（补充）：`executePageLoadActionsSaga`（`PluginActionSaga.ts:1290-1339`，按依赖分批并行执行 on-page-load actions/JS）；求值循环本身会产出 `executeReactiveActions`（`DataTreeEvaluator/index.ts:1214-1245`：JS 函数/动作的依赖变化时自动入队，feature flag 下由 `EvaluationsSaga.ts:325-338` 派 `EXECUTE_REACTIVE_QUERIES` 执行）。

---

## 4. 求值引擎细节

### 4.1 为什么用 Web Worker

整树 diff（deep-diff/micro-diff）、依赖图重排、大量 `eval` 都是 CPU 密集，放 worker 避免阻塞 UI。此外还有第二个 worker：`workers/Tern/tern.worker.ts`（代码补全/lint，独立进程；defs 在 55-73 行：ecmascript/lodash/moment/forge/base64 + Appsmith GLOBAL_FUNCTIONS）。求值后主线程把求值树转 Tern defs：`sagas/PostEvaluationSagas.ts:234-289`（`dataTreeTypeDefCreator` → `CodemirrorTernService.updateDef("DATA_TREE", ...)`）。

### 4.2 Worker action 入口对照表

| 概念名 | 实际代码入口 |
|---|---|
| 求整树 | `handlers/evalTree.ts`（EVAL_TREE） |
| `evalTreeDynamicChanges`（旧名） | **`evalTreeWithChanges`**：`workers/Evaluation/evalTreeWithChanges.ts:43` |
| `evaluateDynamicTrigger`（旧名） | **EVAL_TRIGGER**：`handlers/evalTrigger.ts` + saga `evaluateAndExecuteDynamicTrigger`（`EvaluationsSaga.ts:452`） |
| `evaluateSnippet`（旧名） | **EVAL_EXPRESSION**：`handlers/evalExpression.ts`（对 evalTree 跑任意表达式，用于 action selector 字段，入口 `EvaluationsSaga.ts:1065-1109`） |
| 其他 | `handlers/index.ts:33-45`（VALIDATE_PROPERTY / UNDO / REDO / CLEAR_CACHE / SETUP / UPDATE_ACTION_DATA / INIT_FORM_EVAL 等） |

### 4.3 `{{}}` 里能写什么

- **数据字段**：任意同步 JS 表达式；出现 Promise/API 调用会报 `ActionInDataFieldErrorModifier`（`evaluate.ts:431-445`）
- **触发字段**（onClick 等）：可 `await` 的异步语句
- 模板类型见 `evaluate.ts:38-86`：EXPRESSION / ANONYMOUS_FUNCTION / ASYNC_ANONYMOUS_FUNCTION / TRIGGERS / OBJECT_PROPERTY

### 4.4 沙箱模型（重要：没有真正的 JS 沙箱库）

- `indirectEval.ts:1-5`：`(1, eval)(script)` 间接 eval —— 只在全局作用域跑、拿不到闭包变量
- **隔离靠"跑在专用 Web Worker 里"** + worker 内的全局清洗：
  - `resetWorkerGlobalScope`（`evaluate.ts:106-127`）删除用户变量泄漏
  - `domApis.ts` 限制 DOM API
  - `fns/overrides` 拦截 fetch/interval/localStorage/console

### 4.5 解析器（不是 esprima/recast 直接解析 `{{}}`）

- **JS Object body 解析**用 `@shared/ast` 包（`workers/Evaluation/JSObject/index.ts:6` `parseJSObject`），提取函数/变量/参数；函数体转字符串存树（296-305 `convertJSFunctionsToString`），真实函数缓存在 `JSObject/Collection.ts`
- **`{{}}` 本身的拆分**是 `DynamicBindingUtils.ts:40-87` 的手写扫描（括号配对）

---

## 5. 动态路径表语义

定义与工具：`utils/DynamicBindingUtils.ts:176-285`（`DynamicPath {key}`、`isPathADynamicBinding`:205、`isPathDynamicTrigger`:243 等）。

| 路径表 | 语义 | 写入侧 |
|---|---|---|
| `dynamicBindingPathList` | 哪些属性是**数据绑定**（每轮求值成值） | widget：属性写入 saga；Action（API/Query 配置里的 `{{}}`，如 body/path）：`getDynamicBindingsChangesSaga`（`DynamicBindingUtils.ts:527-658`） |
| `dynamicTriggerPathList` | 哪些属性是**事件绑定**（只在事件发生时求值） | widget：属性面板配置的 `triggerPaths`（`entities/Widget/utils.ts:271+`）；Action 的 run/clear 固定（`dataTreeAction.ts:81`） |
| `dynamicPropertyPathList` | 该字段当前处于 **JS 模式**（UI 状态标记） | `WidgetOperationSagas.tsx:526-537` |

求值时的用法：`DataTreeEvaluator.evaluateTree`（1206-1251）—— `isADynamicBindingPath && !isATriggerPath && isDynamicValue(value)` 才求值；`isATriggerPath` 只用于 reactive actions 检测。

**值从字符串变实际值的流程**：`getDynamicValue`（拆段）→ `evaluateSync`（eval）→ 校验解析（`validateAndParseWidgetProperty`）→ 写 evalTree → diff 回主线程。

---

## 6. 三类数据源

| 数据源 | 前端实体 | 树构建 | 执行位置 |
|---|---|---|---|
| **Action（API/Query）** | `entities/Action/`，API 层 `api/ActionAPI.tsx`（v1/actions CRUD + execute） | `ce/entities/DataTree/dataTreeAction.ts`（config=jsonPathKeys 对应的 actionConfiguration、data、isLoading、responseMeta） | **只在后端**（插件执行器 PluginExecutor） |
| **Datasource** | `entities/Datasource/`；后端 `server/.../datasources/` | **不进 DataTree**，是 Action 的配置来源 | — |
| **JS Object（ActionCollection）** | `entities/JSCollection/`；后端 `server/.../actioncollections/` 只做存储/CRUD | `ce/entities/DataTree/dataTreeJSAction.ts` | **纯前端 worker**（body 经 `@shared/ast` 解析，函数以真实 Function 形式驻留 `JSObject/Collection.ts`，经 `getJSActionForEvalContextMap.ts:5-12` 挂到求值上下文） |

结果回填：`updateActionData` → `handlers/updateActionData.ts` → `evalTreeWithChanges`（见生命周期二阶段 C）。JS Object 变量更新走 `JSVariableUpdates.ts:66`（同样用 EVAL_TREE_WITH_CHANGES）。

---

## 7. 依赖图与增量重算

### 7.1 数据结构

`entities/DependencyMap/index.ts`（`dependencies` 正向 / `inverseDependencies` 反向）+ 拓扑工具 `DependencyMapUtils.ts`（`makeParentsDependOnChildren` —— 父路径依赖子路径，保证 `Table1.selectedRow` 变则引用 `Table1` 的表达式也重算）。

### 7.2 建图与增量更新

- **全量建图**：`workers/common/DependencyMap/index.ts:45-141`（`createDependencyMap`：对每个实体的每条绑定 `extractInfoFromBindings` 提取引用路径；结果可用 `AppComputationCache` 缓存到 IndexedDB 加速首屏）
- **增量更新**：同文件 `updateDependencyMap:169-419`（按 diff 的 NEW/DELETE/EDIT 增删节点与依赖；删除路径时收集 `dependenciesOfRemovedPaths` 让依赖者也重算）

### 7.3 重算范围计算

| 步骤 | 位置 | 作用 |
|---|---|---|
| `setupUpdateTree` | `DataTreeEvaluator/index.ts:666-818` | diff → 改依赖图 |
| `setupTree` → `calculateSubTreeSortOrder` / `getCompleteSortOrder` | `:1952+` / `:1007-1065` | BFS 沿 `inverseDependencies` 扩散 + 用全局 `sortedDependencies` 拓扑序过滤，保证求值顺序正确 |
| `getEvaluationOrderAndSetEvalTreeWithNewUnevalTreeValues` | `:820-850` | 只保留 dynamic leaf |

### 7.4 widget 内部属性依赖

`WidgetFactory.getWidgetDependencyMap`（如 Table 的 primaryColumns→derivedColumns）进入 configEntity.dependencyMap（`dataTreeWidget.ts:207, 354`），由 `DependencyMapUtils` 合并入全局图。反向依赖也回传主线程供 UI 使用（`EvaluationsSaga.ts:319` `setDependencyMap`；selector `getEvaluationInverseDependencyMap`，`dataTreeSelectors.ts:202-203`）。

---

## 8. 易混淆点备注

1. **求值后的值对同一轮后续表达式立即可见**（`contextTree` 即被 eval 的树，`DataTreeEvaluator/index.ts:1144-1150` 注释），但发给主线程的是 `safeTree` 增量 diff。
2. `appsmith.store`（storeValue）走 `PROCESS_STORE_UPDATES` → `StoreActionSaga`（`EvalWorkerActionSagas.ts:143-146`），再触发求值。
3. 求值循环有专门缓冲 `evalQueueBuffer`（`EvaluationsSaga.ts:679-773`），把 `UPDATE_ACTION_DATA`（动作数据回填）与普通求值 action 合并/去抖，**避免一轮双算**（731-746、1026-1046）。
4. cyclical dependency 时求值器会重建整棵树（`evalTree.ts:141-193`）。
5. widget setter（如 `Table1.setVisibility()`）也在 worker 内执行并触发重算：`workers/Evaluation/setters.ts:36-140`（改 evalTree + self + `evalTreeWithChanges`）。

---

## 9. 设计要点小结（可借鉴的抽象）

1. **双树分离**：未求值树（事实）与配置树（元数据）分离，求值树在 worker 内第三份 —— 职责清晰，diff/依赖计算各得其所。
2. **依赖图驱动增量求值**：全量建图一次，之后 diff → 改图 → 反向扩散算出最小重算集，拓扑序保证正确性 —— 大应用不卡的关键。
3. **worker 内 eval + 主线程副作用**：表达式在 worker 求值，但 `run()/navigateTo()` 等副作用以"描述符"发回主线程执行（Promise 回填）—— **计算与副作用分离**，副作用仍受 Redux/网络层管控。
4. **动态路径表作为"求值索引"**：只扫表不扫全树，属性配置（isBindProperty/isTriggerProperty）与求值引擎通过这张表解耦。
5. **模板替换策略可配置**（TEMPLATE/SMART_SUBSTITUTE/PARAMETER）：同一套求值管线适配字符串拼接、对象插值、SQL 预编译三种场景。
6. **去抖缓冲合批**（evalQueueBuffer）：把数据回填与属性变更合并成一轮求值，避免重复计算。
