# Appsmith 源码分析（四）：属性如何配置 —— Property Pane 机制

> 本文梳理右侧属性面板（Property Pane）的声明式配置体系、控件分发机制、属性写入数据流，以及动态绑定/派生属性/校验等特殊机制。
> 所有结论均标注代码位置（`文件路径:行号`），行号基于当前 release 分支。

---

## 1. 总览：属性面板是"声明式配置 + 统一控件注册表 + 集中式写入通道"

业务上一句话：**每个 Widget 用一份静态配置描述自己的属性面板长什么样；面板控件统一注册、按 `controlType` 分发渲染；任何属性修改都收敛到同一条 Redux 通道，原子地写回 DSL 并触发求值重算。**

```
┌─────────────────────────────────────────────────────────────┐
│ Widget 类静态方法 getPropertyPaneContentConfig()/StyleConfig │  ← 声明
│         (section → property → controlType + 校验/绑定标记)   │
└──────────────────────────┬──────────────────────────────────┘
                           │ WidgetFactory 加工（增强/序列化/生成 id）
┌──────────────────────────▼──────────────────────────────────┐
│ PropertyPane → PropertyPaneView → PropertyControlsGenerator  │  ← 渲染
│   → PropertySection / PropertyControl                        │
│     → PropertyControlFactory.createControl(controlType)      │  ← 分发
│       → propertyControls/ 下 60+ 个 BaseControl 子类          │
└──────────────────────────┬──────────────────────────────────┘
                           │ 用户修改属性
┌──────────────────────────▼──────────────────────────────────┐
│ BATCH_UPDATE_MULTIPLE_WIDGETS_PROPERTY action                │  ← 写入
│   → WidgetOperationSagas（维护 dynamicBindingPathList 等表）  │
│     → UPDATE_LAYOUT → canvasWidgetsReducer（DSL 更新）        │
│       → EvaluationsSaga → Web Worker 求值（{{}} 重算）        │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. propertyPaneConfig：声明层

### 2.1 声明位置

属性面板配置**不是独立文件**，而是 Widget 类上的静态方法，定义在基类 `BaseWidget`：

| 静态方法 | 位置 | 职责 |
|---|---|---|
| `getPropertyPaneConfig()` | `app/client/src/widgets/BaseWidget.tsx:120` | 旧版合并配置（Content+Style 合一） |
| `getPropertyPaneContentConfig()` | `BaseWidget.tsx:120-130` | **Content 标签页**配置（业务属性） |
| `getPropertyPaneStyleConfig()` | 同上 | **Style 标签页**配置（外观属性） |
| `getSetterConfig()` | `BaseWidget.tsx:116` | JS setter API（如 `Button1.setVisible()`） |
| `getDerivedPropertiesMap()` | `BaseWidget.tsx:132` | 派生属性模板 |
| `getDefaultPropertiesMap()` | `BaseWidget.tsx:138` | 默认值属性映射 |
| `getDependencyMap()` | `BaseWidget.tsx:142` | 属性间依赖 |
| `getMetaPropertiesMap()` | `BaseWidget.tsx:149` | meta 属性 |
| `getStylesheetConfig()` | `BaseWidget.tsx:153` | 主题样式表 |

每个 Widget 的目录结构为 `src/widgets/<Name>Widget/`：
- `widget/index.tsx`：配置 + 逻辑（静态方法都写在这里）
- `component/index.tsx`：纯渲染组件
- `index.ts`：导出

示例入口：`app/client/src/widgets/ButtonWidget/widget/index.tsx:153` 的 `static getPropertyPaneContentConfig()`。

### 2.2 数据结构（只有两层：Section → Property）

类型全部集中在 `app/client/src/constants/PropertyControlConstants.tsx`：

```ts
// PropertyControlConstants.tsx:456-458
PropertyPaneConfig = PropertyPaneSectionConfig | PropertyPaneControlConfig
```

> 注意：**没有独立的 "category" 层**。"分类"由两种方式表达：
> 1. 面板顶部 **Content / Style 双标签页**（`PropertyPaneTab.tsx`，分别对应 contentConfig / styleConfig 两组配置）
> 2. `panelConfig` **嵌套子面板**（表格列编辑器、Tabs 的 tab 列表等）

**Section 层**（`PropertyPaneSectionConfig`，`PropertyControlConstants.tsx:19-104`）：

| 字段 | 业务含义 |
|---|---|
| `sectionName` | 折叠分组标题（如 "Basic" / "General"） |
| `id?` | 唯一 id，WidgetFactory 启动时自动生成，用于 Redux 记忆折叠状态与 React key |
| `children: PropertyPaneConfig[]` | 组内属性控件（可再嵌套 section） |
| `collapsible` / `isDefaultOpen` | 是否可折叠 / 默认展开 |
| `hidden(props, propertyPath)` | 运行时回调决定整组隐藏 |
| `hasDynamicProperties` + `generateDynamicProperties(widget)` | 运行时动态生成属性项（如 CustomWidget 按事件动态生成） |

**Property 层**（`PropertyPaneControlConfig`，`PropertyControlConstants.tsx:125-331`）核心字段：

| 字段 | 业务含义 |
|---|---|
| `propertyName` | 属性路径，支持嵌套（如 `primaryColumns.foo.isVisible`） |
| `label` / `helpText` / `helperText` | 标签 / tooltip 提示 / 输入框下方动态说明 |
| `controlType: ControlType` | 控件类型字符串，取自控件注册表 |
| `isBindProperty` | 是否允许 `{{}}` 绑定（参与生成 bindingPaths） |
| `isTriggerProperty` | 是否事件触发属性（onClick 等，进 `dynamicTriggerPathList`） |
| `isJSConvertible` | label 旁是否显示 JS 模式切换按钮 |
| `validation: ValidationConfig` | 校验规则 `{type, params}` |
| `hidden(props, path)` / `invisible` | 两种隐藏方式（回调 / 布尔） |
| `updateHook(props, name, value)` | 改本属性时联动改**本 widget 其他属性**，返回 `PropertyUpdates[]` |
| `updateRelatedWidgetProperties(...)` | 联动改**其他 widget** 的属性 |
| `dependencies` 等三个依赖字段 | 声明依赖的其他属性，注入控件 `widgetProperties` |
| `options` / `placeholderText` / `defaultValue` / `expected` | 下拉选项 / 占位符 / 默认值 / CodeEditor 类型提示 |
| `dataTreePath` | 取 evaluated（求值后）值的路径 |
| `customJSControl` | JS 模式下替换渲染的控件（如表格的 `COMPUTE_VALUE`） |
| `panelConfig: PanelConfig` | 嵌套属性面板（`PropertyControlConstants.tsx:106-123`） |
| `postUpdateAction` | 更新完后要 dispatch 的 Redux action |
| `isReusable` | 拖拽新 widget 时复用上次值（存 sessionStorage，`pages/Editor/PropertyPane/helpers.tsx:151`） |
| `evaluationSubstitutionType` | 求值替换策略：`TEMPLATE` / `SMART_SUBSTITUTE` / `PARAMETER`（`constants/EvaluationConstants.ts:1-5`） |
| `getStylesheetValue` | 主题（Theming）取值 |

### 2.3 真实示例（摘自 ButtonWidget，手写精简）

```ts
// app/client/src/widgets/ButtonWidget/widget/index.tsx:153
static getPropertyPaneContentConfig() {
  return [
    {
      sectionName: "Basic",            // Section（折叠分组）
      children: [
        {                              // Property（一个属性控件）
          propertyName: "text",
          label: "Label",
          helpText: "Sets the label of the button",
          controlType: "INPUT_TEXT",   // 由控件注册表分发
          placeholderText: "Submit",
          isBindProperty: true,        // 允许 {{}} 绑定
          isTriggerProperty: false,
          validation: { type: ValidationTypes.TEXT },
        },
        {
          propertyName: "onClick",
          controlType: "ACTION_SELECTOR",
          isJSConvertible: true,       // 显示 JS 切换按钮
          isTriggerProperty: true,     // 事件触发属性
        },
      ],
    },
  ];
}
```

### 2.4 配置加工管道（WidgetFactory）

`app/client/src/WidgetProvider/factory/index.tsx:339-463`：`getWidgetPropertyPaneContentConfig/StyleConfig/Config` 均加了 `@memoize @freeze`，并对原始配置执行流水线
`flow([enhancePropertyPaneConfig, convertFunctionsToString, addPropertyConfigIds, addSearchConfigToPanelConfig])`（helpers 见 `app/client/src/WidgetProvider/factory/helpers.ts`）：

| 步骤 | 位置 | 作用 |
|---|---|---|
| `enhancePropertyPaneConfig` | `helpers.ts:191` | 按 `getFeatures()`（如 auto-height）向 "General" section 注入通用控件 |
| `convertFunctionsToString` | `helpers.ts:273` | `validation.fn` → `fnString`，**保证可序列化传给 Web Worker** |
| `addPropertyConfigIds` | `helpers.ts:130` | 递归生成 `id`（React key），处理 panelConfig 子面板 |
| `addTabOrderToPropertyPaneConfig` | — | 注入 Accessibility 的 tabOrder 属性 |
| `getWidgetPropertyPaneSearchConfig` | `index.tsx:467` | 合并 content+style 生成搜索用扁平配置 |

---

## 3. 属性控件体系：渲染层

### 3.1 控件注册表

- **实现目录**：`app/client/src/components/propertyControls/`（60+ 个控件：InputTextControl、DropDownControl、SwitchControl、CodeEditorControl、ColorPickerControl、DatePickerControl、ActionSelectorControl、OptionControl、TabControl、PrimaryColumnsControl、NumericInputControl、OneClickBindingControl 等）
- **注册表**：`app/client/src/components/propertyControls/index.ts:87-142` 的 `PropertyControls` 对象汇总全部控件；`getPropertyControlTypes()`（`index.ts:174`）遍历每个控件的静态 `getControlType()` 生成 `controlType 字符串` 的映射，即 `ControlType` 类型（`constants/PropertyControlConstants.tsx:15-17`）。企业版控件通过 `EEPropertyControls` 合并（`index.ts:85,141`）

### 3.2 基类 BaseControl

`app/client/src/components/propertyControls/BaseControl.tsx`：

- 每个控件是 class 组件，继承 `BaseControl<P extends ControlProps>`，必须实现静态 `getControlType()`（如 `InputTextControl.tsx:151` 返回 `"INPUT_TEXT"`）
- `updateProperty(propertyName, value)`（`BaseControl.tsx:29`）：去重后调 `this.props.onPropertyChange(...)` —— **所有控件最终通过统一回调上行**
- `batchUpdateProperties` / `batchUpdatePropertiesWithAssociatedUpdates` / `deleteProperties`：批量写入通道
- 静态 `canDisplayValueInUI(config, value)`：JS 模式下该值能否切回 UI 模式（决定 JS toggle 是否禁用）
- `ControlProps = ControlData + ControlFunctions`（`BaseControl.tsx:104-154`）：框架注入 `propertyValue / evaluatedValue / widgetProperties / parentPropertyName` 及回调

### 3.3 工厂与按类型分发

| 组件 | 位置 | 职责 |
|---|---|---|
| `PropertyControlFactory` | `app/client/src/utils/PropertyControlFactory.tsx:25-97` | 维护 3 个静态 Map（控件构建器 / 控件方法 / inputComputedValue）。`createControl()`（44 行）分发：JS 模式开启（`preferEditor`）→ `customJSControl` 指定控件或退回 `CODE_EDITOR`；否则 → `controlMap.get(controlType)` |
| `PropertyControlRegistry` | `app/client/src/utils/PropertyControlRegistry.tsx:67-93` | 遍历 `PropertyControls`，用 `withAnalytics` HOC 包装后注册。**懒加载 chunk**，由 PropertyPane 挂载时动态 import（`pages/Editor/PropertyPane/index.tsx:35-50`，注册完成前不渲染面板） |

### 3.4 渲染组件层级

```
PropertyPaneSidebar (components/editorComponents/PropertyPaneSidebar.tsx)
 └ PropertyPane (pages/Editor/PropertyPane/index.tsx) —— Blueprint PanelStack
    └ PropertyPaneView (pages/Editor/PropertyPane/PropertyPaneView.tsx:79)
       ├ PropertyPaneTitle          # widget 名 / 复制 / 删除
       ├ PropertyPaneSearchInput    # 搜索（propertyPaneSearch.ts 过滤）
       ├ PropertyPaneTab            # Content / Style 双标签页
       └ PropertyControlsGenerator (PropertyControlsGenerator.tsx:44-102)
          ├ 有 sectionName → <PropertySection>   # 可折叠；状态存 Redux（key=widgetId.sectionId）
          └ 有 controlType → <PropertyControl>   # PropertyControl.tsx:90
             └ PropertyControlFactory.createControl (PropertyControl.tsx:1131-1148)
                └ 具体控件（如 InputTextControl → LazyCodeEditor）
                └ 有 panelConfig 时 openPanel → PanelPropertiesEditor（嵌套面板）
```

搜索模式：`searchPropertyPaneConfig`（`pages/Editor/PropertyPane/propertyPaneSearch.ts`）对配置树做匹配与高亮，结果以扁平 PropertyControl 列表渲染。

---

## 4. 属性修改的完整数据流（核心链路）

以「用户在 InputText 属性控件输入文字」为例：

| 步骤 | 环节 | 代码位置 |
|---|---|---|
| 1 | 控件捕获：`InputTextControl.onTextChange` → `updateProperty` | `components/propertyControls/InputTextControl.tsx:141-149` → `BaseControl.tsx:29-46` |
| 2 | `PropertyControl.onPropertyChange` 组装 payload | `pages/Editor/PropertyPane/PropertyControl.tsx:571-630`；先经 selector `getWidgetPropsForPropertyName` 取当前值（`selectors/propertyPaneSelectors.tsx:220-256`） |
| 2a | └ `getWidgetPropsForPropertyName` 内调 `getWidgetsOwnUpdatesOnPropertyChange`：先写入本次修改，再执行配置里的 `updateHook`，合并为 `{modify, remove, postUpdateAction}` + `dynamicUpdates.dynamicPropertyPathList` | `PropertyControl.tsx:292-400` |
| 2b | └ `getOtherWidgetPropertyChanges`：执行 `updateRelatedWidgetProperties` 与 enhancement（如 List/Table 子 widget 联动） | `PropertyControl.tsx:402-505` |
| 3 | dispatch `batchUpdateMultipleWidgetProperties(updatesArray)`（type=`BATCH_UPDATE_MULTIPLE_WIDGETS_PROPERTY`）。**原子合并多 widget 更新，保证 undo/redo 快照一致**（`PropertyControl.tsx:507-564` 按 widgetId merge） | `actions/controlActions.tsx:53-60` |
| 4 | saga `batchUpdateMultipleWidgetsPropertiesSaga` 处理 | `sagas/WidgetOperationSagas.tsx:839-892` |
| 4a | └ `getPropertiesUpdatedWidget` → `computeWidgetProperties` → `getPropertiesToUpdate`：**在这里维护动态路径表** —— 对每个更新的字符串值用 `isDynamicValue({{}})` 检测，结合 `getAllPathsFromPropertyConfig` 得到的 `triggerPaths`（`entities/Widget/utils.ts:253-371`），分别 ADD/REMOVE 到 `dynamicTriggerPathList` / `dynamicBindingPathList`，再 `purgeOrphanedDynamicPaths` 清理孤儿路径 | `WidgetOperationSagas.tsx:635-799` |
| 4b | └ lodash `set(widget, propertyPath, value)` 写入 widget 对象（支持嵌套路径） | 同上 |
| 5 | `updateAndSaveLayout(widgets, ...)`（type=`UPDATE_LAYOUT`）→ reducer 按 diff 或 `updatedWidgetIds` 替换 `canvasWidgets[widgetId]`。**这就是 DSL 里 widget.props 的更新点**；持久化由页面保存流程完成（见《02-页面保存》） | `actions/pageActions.tsx:183-193` → `ce/reducers/entityReducers/canvasWidgetsReducer.ts:74-112` |
| 6 | `UPDATE_LAYOUT` 触发求值：EvaluationsSaga 派发 EVAL_TREE 到 Web Worker，`DataTreeEvaluator` 按依赖拓扑序执行 `{{}}` 求值 | `ce/actions/evaluationActionsList.ts:29,43` → `workers/Evaluation/handlers/evalTree.ts:48` |
| 7 | 回读显示：求值结果写入 evaluated data tree；`getWidgetPropsForPropertyPane` 把 `evaluatedWidget.__evaluation__` 挂到 props；PropertyControl 用 `getEvalValuePath(dataTreePath)` 取 `evaluatedValue` 展示"求值结果"弹层 | `selectors/propertyPaneSelectors.tsx:72-108`、`PropertyControl.tsx:767-773` |

其他写入 action（`app/client/src/actions/controlActions.tsx`）：

| Action | 位置 | 说明 |
|---|---|---|
| `updateWidgetPropertyRequest` | `:6` | 单属性写入（saga 内部转发为 batch） |
| `batchUpdateWidgetProperty` | `:30` | **经 redux-saga `actionChannel` 串行化消费**（`WidgetOperationSagas.tsx:1934-1952`），保证状态串行 flush |
| `deleteWidgetProperty` | `:62` | 转为 `{remove: paths}` 的 batch（saga:933-944） |
| `setWidgetDynamicProperty` | `:73` | JS 模式切换专用 |
| `batchUpdateWidgetDynamicProperty` | `:43` | 批量 JS 模式切换 |

---

## 5. 动态属性 / `{{}}` 绑定

### 5.1 表达式检测

- 正则：`DATA_BIND_REGEX = /{{([\s\S]*?)}}/`（`app/client/src/constants/BindingsConstants.ts:1`）
- 判定函数：`isDynamicValue(value)`（`app/client/src/utils/DynamicBindingUtils.ts:36`）

### 5.2 三个动态路径表（存储在 DSL 上，每个 widget 一份）

```ts
// app/client/src/utils/DynamicBindingUtils.ts:176-185
interface DynamicPath { key: string; value?: string }

interface WidgetDynamicPathListProps {
  dynamicBindingPathList?: DynamicPath[];  // 需要求值的 {{}} 绑定属性
  dynamicTriggerPathList?: DynamicPath[];  // 事件触发属性（onClick 等，惰性执行不求值）
  dynamicPropertyPathList?: DynamicPath[]; // 被用户切到"JS 模式"的属性
}
```

- 每次属性写入时由 `getPropertiesToUpdate` **增量维护**（`getDynamicBindingPathListUpdate` / `getDynamicTriggerPathListUpdate` + `applyDynamicPathUpdates`，`WidgetOperationSagas.tsx:448-461`）
- 事件属性判定来源：propertyPaneConfig 里 `isTriggerProperty: true` 的控件 → `getAllPathsFromPropertyConfig` 的 `triggerPaths`（`entities/Widget/utils.ts:271`）

### 5.3 UI 入口与 JS 模式切换

- **JS toggle 按钮**：`isJSConvertible` 才显示（`PropertyControl.tsx:1013-1042`）；值无法用 UI 表达时禁用（`canDisplayValueInUI`，853-887 行）
- 点击 → `toggleDynamicProperty`（`PropertyControl.tsx:216-261`）→ `setWidgetDynamicProperty` action → `setWidgetDynamicPropertySaga`（`WidgetOperationSagas.tsx:597-633`）→ `handleUpdateWidgetDynamicProperty`（511-577 行）：
  - **切到 JS**：push 进 `dynamicPropertyPathList`，值 `convertToString`
  - **切回普通**：从 `dynamicPropertyPathList` 移除；若 `shouldRejectDynamicBindingPathList`（COLOR_PICKER 等除外，`PropertyControl.tsx:87`）则从 `dynamicBindingPathList` 移除该路径，并调 `validateProperty`（`EvaluationsSaga.ts:616-638` → worker `EVAL_WORKER_ACTIONS.VALIDATE_PROPERTY`，`workers/Evaluation/handlers/validateProperty.ts`）把 JS 值解析成普通模式可用值（失败则用 validation 返回的默认值）
- **其他绑定入口**：CodeEditor 内直接写 `{{}}`（无需切换，saga 自动加入 bindingPathList）；`ACTION_SELECTOR`（事件面板）；一键绑定 `OneClickBindingControl`；调试器/属性检查器（State Inspector）；表格列的 `customJSControl: "COMPUTE_VALUE"`
- **求值侧**：构建 data tree 时遍历 `dynamicBindingPathList`（`entities/DataTree/dataTreeWidget.ts:213-238`），对象值先 stringify；每个绑定路径按该控件的 `evaluationSubstitutionType` 在 worker 中以 TEMPLATE / SMART_SUBSTITUTE / PARAMETER 三种方式替换

---

## 6. 特殊属性机制

### 6.1 derivedProperties（派生属性）

- 声明：`static getDerivedPropertiesMap()`（`BaseWidget.tsx:132`），值是含 `{{this.xxx}}` 的模板字符串。例：`InputWidget` 的 `isValid`（整段校验函数）、`value: "{{this.text}}"`（`app/client/src/widgets/InputWidget/widget/index.tsx:752-828`）
- 处理：`dataTreeWidget.ts:226-238` —— 把 `this.` 替换为 `widgetName.`，并**追加进 dynamicBindingPathList** 参与求值；派生属性不报 lint 错误（`blockedDerivedProps`）
- **业务解释：派生属性 = 随依赖自动重算的只读绑定属性**，结果放在 `__evaluation__` 下

### 6.2 defaultProperties（默认值属性）

- 声明：`static getDefaultPropertiesMap()`（`BaseWidget.tsx:138`），映射 `显示属性 → 默认属性`。例：`InputWidget:830-834` 的 `{ text: "defaultText" }`
- 处理：`dataTreeWidget.ts:248-269` —— 求值时 `defaultText` 的值会**覆盖** `text`（叠加 meta 覆盖优先级）；运行态组件读 `text` 拿到的就是"用户输入或默认值"。defaultProperties 同时作为 reactivePaths 注册（`entities/Widget/utils.ts:268-270`）
- **业务解释：解决"运行时用户输入"与"设计时默认值"的同一化** —— 组件代码只读 `text`，无需关心值来自哪

### 6.3 hiddenProperties（注意：不存在此字段名）

隐藏由两层实现：
1. 配置项 `hidden(props, path)` 回调 / `invisible` 布尔（`PropertyControlConstants.tsx:217-240`；渲染判定在 `PropertyControl.tsx:740-748` 与 PropertyControlsGenerator 的 `evaluateHiddenProperty`，`helpers.tsx:34`）
2. **重要副作用**：`getAllPathsFromPropertyConfig`（`entities/Widget/utils.ts:284-301`）只把**未隐藏**的控件属性注册进 bindingPaths / validationPaths —— **隐藏的属性不参与绑定求值与校验**

### 6.4 setterConfig（widget 的 JS setter API）

- 声明：`static getSetterConfig()`（`BaseWidget.tsx:116`）；类型 `app/client/src/entities/AppTheming/index.ts:83-92`，形如 `{ __setters: { setVisibility: { path: "isVisible", type: "boolean" } } }`
- 例：`ButtonWidget:638-659`（`setLabel→text`、`setColor→buttonColor` 等）
- **业务解释：让 JS 里可写 `Button1.setVisible(false)` 直接改属性**。生成于 `dataTreeWidget.ts:140-174` 与 worker 的 `workers/Evaluation/setters.ts` / `evaluate.ts:520`（`shouldAddSetter`）—— setter 触发 `UPDATE_WIDGET_PROPERTY`，**走与属性面板完全相同的 Redux 通道**

### 6.5 validation（属性校验）

- 类型枚举：`app/client/src/constants/WidgetValidation.ts:5-20` —— TEXT / REGEX / NUMBER / BOOLEAN / OBJECT / ARRAY / OBJECT_ARRAY / NESTED_OBJECT_ARRAY / DATE_ISO_STRING / IMAGE_URL / FUNCTION / SAFE_URL / ARRAY_OF_TYPE_OR_TYPE / UNION / OBJECT_WITH_FUNCTION
- 配置结构：`ValidationConfig {type, params}`（`PropertyControlConstants.tsx:333-454`；params 含 min/max/regex/allowedKeys/allowedValues/children/fn/unique/required 等，UNION 支持递归）
- **执行位置在 Web Worker 内**（`workers/Evaluation/validations.ts` + `handlers/validateProperty.ts`），两处触发：
  1. 求值树构建时对绑定值校验（validationPaths 来自 `entities/Widget/utils.ts:274,300`）
  2. JS → 普通模式切换时（`EvaluationsSaga.ts:616` → `WidgetOperationSagas.tsx:561-570`）
- 校验错误通过 `__evaluation__.errors` 回流到控件显示（`propertyPaneSelectors.tsx:184-208`）

### 6.6 批量更新优化（原子性与串行化）

- 单次用户操作的多 widget / 多属性改动合并为**一个** `BATCH_UPDATE_MULTIPLE_WIDGETS_PROPERTY`，saga 里 `yield all(...)` 并行计算后一次 `updateAndSaveLayout`（`WidgetOperationSagas.tsx:839-892`）—— **保证 undo/redo 快照原子**
- `BATCH_UPDATE_WIDGET_PROPERTY` 用 `actionChannel` 串行消费（`WidgetOperationSagas.tsx:1934-1952`）
- `updateHook` 返回 `PropertyUpdates[]`（`WidgetProvider/types.ts:212-217`，支持 `shouldDeleteProperty` / `isDynamicPropertyPath`）是"一次交互改多属性"的配置化手段（如 Tabs 改 label 同步 tabs 数组）

### 6.7 其他相关机制

| 机制 | 位置 | 业务解释 |
|---|---|---|
| 主题样式 | `getStylesheetConfig` + 控件级 `getStylesheetValue`；偏离主题值时显示"重置"按钮 | 属性值可来自主题 token，改过则可一键还原（`PropertyControl.tsx:164-203, 1043-1057`） |
| 属性复用 | `isReusable: true` 的属性值存 sessionStorage | 下次拖同类型 widget 复用上次值（`PropertyPane/helpers.tsx:151`） |
| 动态属性生成 | section 的 `hasDynamicProperties / generateDynamicProperties` | CustomWidget 按模型事件动态生成控件（`factory/index.tsx:407-431`） |
| EE 扩展 | `ee/components/propertyControls` 注入注册表 | 企业版专属控件与 feature flag |

---

## 7. 设计要点小结（可借鉴的抽象）

1. **声明式面板配置**：属性面板完全由数据描述（section → property → controlType），widget 作者不写 UI 代码；配置可被 memoize/freeze/序列化到 worker。
2. **控件注册表 + 工厂分发**：新控件只需继承 `BaseControl` + `getControlType()` 注册，面板与控件解耦；JS 模式通过 `preferEditor + customJSControl` 在工厂层切换，不影响控件实现。
3. **统一写入通道**：所有修改（面板输入、JS setter、代码联动）收敛到 batch action，天然支持原子 undo/redo 与串行化。
4. **动态路径表与属性配置联动**：`isBindProperty / isTriggerProperty` 决定值进入哪张表，写入时增量维护，求值只扫表不扫全树。
5. **校验/求值下沉 worker**：函数序列化（`convertFunctionsToString`）是配置能跨线程传输的关键技巧。
