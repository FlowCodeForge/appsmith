# Appsmith 源码分析（三）：组件如何抽象 —— Widget 体系

> 本文梳理 Widget 的目录组织、核心抽象（BaseWidget/WidgetProps）、注册与懒加载机制、容器嵌套与布局系统、编辑器/运行时渲染链路，最后给出「定义一个新 Widget 的清单」。
> 所有结论均标注代码位置（`文件路径:行号`），行号基于当前 release 分支。

---

## 1. 总览：两层分离 + 静态元配置 + 注册表

业务上一句话：**每个 Widget = 一个继承 `BaseWidget` 的类（逻辑/配置层） + 一个纯 UI 组件（表现层）；类的静态 getter 声明全部"元配置"（默认值、属性面板、派生属性、setter…），通过懒加载注册表汇入 `WidgetFactory`，渲染时由工厂按 `type` 生成 HOC 链包装的 React 节点。**

```
widgets/index.ts (WidgetLoaders: type → () => import())
        │  import 副作用注册
        ▼
widgets/registry.ts (loadWidget / loadAllWidgets)
        │  registerWidgets(widgetClasses)
        ▼
WidgetProvider/factory (WidgetFactory: widgetsMap / widgetBuilderMap / widgetConfigMap)
        │  createWidget(dslNode, renderMode) → builder → HOC 链
        ▼
withWidgetProps → withLazyRender → withLayoutSystemWidgetHOC → [withMeta] → BaseWidget 子类
        │  getWidgetView()
        ▼
component/index.tsx（纯 UI 组件，受控、无 Redux、无 DSL 访问）
```

---

## 2. Widget 目录结构与分组

**根目录：`app/client/src/widgets/`**（65 个子目录 + 若干顶层文件）

| 分组 | 位置 | 说明 |
|---|---|---|
| 旧版（Fixed/AutoLayout 布局用）widget | `widgets/` 顶层，如 `ButtonWidget/`、`InputWidgetV2/`、`ContainerWidget/`、`TableWidgetV2/` | 每目录一个 widget |
| Anvil 布局专用 WDS widget | `widgets/wds/`（`WDSButtonWidget/`、`WDSZoneWidget/`、`WDSSectionWidget/` 等约 30 个） | 基于 `@appsmith/wds` 设计系统 |
| 可复用基类 widget | `widgets/BaseInputWidget/` | Input/Currency/Phone 等共享的抽象基类（`BaseInputWidget/widget/index.tsx:27`） |
| 共享组件 | `widgets/components/` | 如 `LabelWithTooltip.tsx` |
| EE 扩展位 | `app/client/src/ee/widgets/index.ts`（`EEWidgets = []`）、`ee/widgets/wds/` | 企业版注入点 |
| Deprecated widget | 仍在顶层（`DropdownWidget`、`InputWidget`、`ListWidget` 等） | `widgets/index.ts:373-400` 标注 "Deprecated Widgets"，保留用于旧 DSL 兼容 |

顶层关键文件：

| 文件 | 职责 |
|---|---|
| `widgets/BaseWidget.tsx` | 所有 widget 的抽象基类 |
| `widgets/BaseComponent.tsx` | 表现层组件抽象基类（仅 17 行，标明 Widget/Component 两层分离的约定） |
| `widgets/index.ts` | 懒加载 loader 总表（barrel） |
| `widgets/registry.ts` | 无静态依赖的 loader 注册表（刻意打断循环依赖，见 `registry.ts:6-21` 注释） |
| `widgets/withWidgetProps.tsx` / `withLazyRender.tsx` / `MetaHOC.tsx` / `BaseWidgetHOC/withBaseWidgetHOC.ts` | widget 的 HOC 层 |
| `widgets/CanvasWidget.tsx` / `SkeletonWidget.tsx` / `WidgetUtils.ts` / `utils.ts` | 画布/骨架屏/工具 |

**单个 widget 目录的标准形态**（如 `ButtonWidget/`）：

```
ButtonWidget/
  index.ts          # re-export ./widget
  widget/index.tsx  # Widget 类（继承 BaseWidget）+ Props 接口 + 默认导出
  component/index.tsx (+ DragContainer.tsx / utils.tsx)  # 纯 UI 组件（React 视图层）
  icon.svg          # 组件面板图标
  thumbnail.svg     # 拖拽预览缩略图
```

---

## 3. 核心抽象

### 3.1 BaseWidget —— widget 的"控制器"基类

`app/client/src/widgets/BaseWidget.tsx:79`：

```ts
abstract class BaseWidget<T extends WidgetProps, K extends WidgetState, TCache>
  extends Component<T, K>
```

类头注释（1-5 行）明确职责：**"Widget 负责接收抽象层输入、解释为可渲染 props、据此生成组件，并负责派发 action / 更新 state 树"**。同时约定（64-74 行）：widget 内不许直接用 context、不许访问 DSL、不许连 Redux。

**静态"元信息" getter（88-158 行，全部可被子类覆写）—— 一个 widget 的完整"元配置面"：**

| 静态方法 | 行号 | 职责 |
|---|---|---|
| `static type` | 88 | widget 类型字符串，如 `"BUTTON_WIDGET"` |
| `getDefaults()` | 90 | `WidgetDefaultProps`：新建 widget 的默认属性（会写进 DSL） |
| `getConfig()` | 94 | 名称/图标/标签/hideCard/isCanvas/needsMeta/eagerRender 等 |
| `getFeatures()` | 100 | 特性开关（如 `dynamicHeight`，`utils/WidgetFeatures.ts:41`） |
| `getMethods()` | 104 | getSnipingModeUpdates / getCanvasHeightOffset / getQueryGenerationConfig 等（`WidgetProvider/types.ts:219`） |
| `getAutoLayoutConfig()` | 108 | AutoLayout（flex）布局下的尺寸/resize 配置 |
| `getAnvilConfig()` | 112 | Anvil 布局下的 min/max 尺寸 |
| `getSetterConfig()` | 116 | JS setter API 定义（如 `Button1.setDisabled()`） |
| `getPropertyPaneConfig()` / `...ContentConfig()` / `...StyleConfig()` | 120/124/128 | 属性面板配置（详见《04-属性配置》） |
| `getDerivedPropertiesMap()` | 132 | `Record<string, string>`，值为 `{{...}}` 绑定表达式，由求值引擎计算 |
| `getDefaultPropertiesMap()` | 138 | metaProp → 静态属性映射（如 `inputText: "defaultText"`，reset 时回落） |
| `getDependencyMap()` | 142 | 属性间依赖 |
| `getMetaPropertiesMap()` | 149 | 运行时可变状态（meta，如 `inputText`、`recaptchaToken`） |
| `getStylesheetConfig()` | 153 | 主题 stylesheet（`{{appsmith.theme...}}`） |
| `getAutocompleteDefinitions()` | 157 | JS 编辑器自动补全定义 |
| `pasteOperationChecks()` / `performPasteOperation()` | 162/172 | 粘贴行为钩子 |
| `getLoadingProperties()` | 198 | 哪些绑定路径能触发 isLoading |

**实例方法：**

| 方法 | 行号 | 职责 |
|---|---|---|
| `abstract getWidgetView(): ReactNode` | 400 | **唯一必须实现的渲染方法**；`render()`（382）直接调它 |
| `executeAction()` | 206 | 触发动作（走 `EditorContext`） |
| `updateWidget / batchUpdateWidgetProperty / updateWidgetProperty / deleteWidgetProperty` | 235-274 | 更新 DSL 属性 |
| `modifyMetaWidgets / deleteMetaWidgets / updateMetaWidgetProperty` | 324-352 | meta widget（List/Tabs 动态生成的子 widget）操作 |
| `shouldComponentUpdate` | 405 | `shallowequal` 比较 |

### 3.2 WidgetProps —— widget 的统一数据形态

`widgets/BaseWidget.tsx:570`：`WidgetProps extends WidgetDataProps, WidgetDynamicPathListProps, DataTreeEvaluationProps`，组合自：

- `WidgetBaseProps`（462）：widgetId/type/widgetName/parentId/renderMode/version/hasMetaWidgets/isMetaWidget 等
- `WidgetRowCols + WidgetPositionProps`（494/507）：topRow/bottomRow/leftColumn/rightColumn、parentRowSpace/parentColumnSpace、detachFromLayout、responsiveBehavior、layoutSystemType 等
- `WidgetDisplayProps`（550）：isVisible/isLoading/isDisabled/backgroundColor/deferRender 等
- 最终带 `[key: string]: any` 索引签名（579）—— **任何 widget 自定义属性都塞进 WidgetProps**

相邻类型（一个 widget 数据的三种形态）：

| 类型 | 位置 | 形态 |
|---|---|---|
| `FlattenedWidgetProps` | `WidgetProvider/types.ts:45` | `WidgetProps & { children?: string[] }` —— **DSL 扁平化存储形态** |
| `DSLWidget` | `WidgetProvider/types.ts:49` | `WidgetProps & { children?: DSLWidget[] }` —— 服务端 DSL 树形态 |
| `CanvasWidgetStructure` | `WidgetProvider/types.ts:78` | 树形渲染形态 |

### 3.3 WidgetType 与 WidgetConfiguration

- `WidgetType`（`constants/WidgetConstants.tsx:5`）= `FactoryWidgetType`（`WidgetProvider/factory/types.ts:6`），本质是 string。**真正的类型集合来自运行时注册**（`WidgetFactory.widgetTypes`），而非枚举 —— "类型 = 注册进 factory 的 `widget.type` 字符串"。
- `WidgetConfiguration`（`WidgetProvider/types.ts:189`）：描述"一个 widget 的完整配置"的聚合接口 —— WidgetFactory 把各静态 getter 摊平后的目标形态，包含 `WidgetBaseConfiguration + autoLayout + defaults + features + properties{config/contentConfig/styleConfig/default/meta/derived/loadingProperties/stylesheetConfig/autocompleteDefinitions/setterConfig} + methods`。
- `WidgetBlueprint`（`WidgetProvider/types.ts:281`）：添加 widget 时自动创建子结构的蓝图（如 Container 自动带一个 `CANVAS_WIDGET` 子节点）。

---

## 4. 注册 / 发现 / 懒加载机制

四层结构（刻意用 registry 打断循环依赖）：

1. **loader 表**：`widgets/index.ts:8` 定义 `WidgetLoaders = new Map<string, () => Promise<typeof BaseWidget>>`，每个 entry 是 `[type, () => import("./XxxWidget").then(m => m.default)]` —— **动态 `import()` 即代码分割点**。
2. **registry**：`widgets/registry.ts:31` `registerWidgetLoaders()`；`widgets/index.ts:406` 在模块被 import 时作为副作用注册。`loadWidget(type)`（`registry.ts:40`，带 retry + 缓存）和 `loadAllWidgets()`（`registry.ts:63`）对外提供。
3. **两个消费端**：
   - 编辑器/主线程：`utils/editor/EditorUtils.ts:6-15` `registerAllWidgets()` 调 `loadAllWidgets()` 全量加载并 `registerWidgets(...)`；`editorInitializer`（17 行）调用它。
   - **求值 worker**：`workers/Evaluation/evaluation.worker.ts:6` `import "widgets"` 填充 registry；`sagas/EvaluationsSaga.ts:916-921` **只加载 DSL 中实际用到且未注册的 widget 类型**再注册（按需加载）。
4. **WidgetFactory**（`WidgetProvider/factory/index.tsx:53`）：
   - `registerWidget`（`registrationHelper.tsx:24`）：读 `widget.getConfig()` 的 `needsMeta/eagerRender`，经 `withBaseWidgetHOC` 包装成可渲染组件。
   - `WidgetFactory.initialize()`（`factory/index.tsx:70`）：写入 `widgetsMap / widgetTypes / widgetBuilderMap`，并 `configureWidget`（91 行）把 defaults+config 摊平进 `widgetConfigMap`、`widgetDefaultPropertiesMap`（并 Object.freeze）。
   - 渲染入口 `createWidget(widgetData, renderMode)`（187 行）：从 `widgetBuilderMap` 取 builder 生成 React 节点。
   - 元信息查询 API：`getWidgetTypes()`（218）、`getWidgetDerivedPropertiesMap()`（224）、`getWidgetDefaultPropertiesMap()`（242）、`getWidgetDependencyMap()`（262）、`getWidgetMetaPropertiesMap()`（280）、属性面板三个 getter（302-463）、`getWidgetAutoLayoutConfig()`（485）、`getWidgetAnvilConfig()`（529）、`getWidgetTypeConfigMap()`（548，供求值引擎）、`getWidgetSetterConfig()`（582）、`getWidgetStylesheetConfigMap()`（600）、`getWidgetMethods()`（617）。
   - 多数 getter 有 `@memoize @freeze`（`factory/decorators.ts`）；**注册新 widget 后需 `clearAllWidgetFactoryCache()`**（`EvaluationsSaga.ts:923`）。
   - 注册后 `incrementWidgetConfigsVersion()`（`registrationHelper.tsx:21` + `factory/widgetConfigVersion.ts`）触发依赖 widget 配置的 selector 刷新（如组件面板 `getWidgetCards`，`selectors/editorSelectors.tsx:445`，按 Anvil/非 Anvil 过滤 widget 列表）。

**HOC 包装链**（`widgets/BaseWidgetHOC/withBaseWidgetHOC.ts:11-29`，flow 从下往上）：

```
withWidgetProps      # 取数：canvas DSL + 求值结果 + meta → 组装最终 props
  → withLazyRender   # IntersectionObserver 视口外先渲染 Skeleton（widgets/withLazyRender.tsx:7）
    → withLayoutSystemWidgetHOC  # 按布局系统包 WidgetWrapper + propertyEnhancer（layoutSystems/withLayoutSystemWidgetHOC.tsx:61）
      → withMeta    # needsMeta=true 时注入 updateWidgetMetaProperty（widgets/MetaHOC.tsx:48）
        → 你的 Widget 类
```

---

## 5. 容器与布局系统

### 5.1 容器如何嵌套子组件

关键机制：**容器型 widget 的默认值里带 `blueprint`，拖入画布时自动生成子 `CANVAS_WIDGET`；渲染时把 children 再交给 `renderAppsmithCanvas` 递归渲染。**

| 容器 | 代码位置 | 机制 |
|---|---|---|
| ContainerWidget | `widgets/ContainerWidget/widget/index.tsx:47` | `getConfig()` 声明 `isCanvas: true`（64 行）；`getDefaults().blueprint.view` 定义子 CANVAS_WIDGET、`blueprint.operations` 在 Anvil 下给该 canvas 注入默认 layout（94-154 行）；`renderChildWidget()`（345 行）调整子 widget 边界后调 `renderAppsmithCanvas(childWidget)`（366 行）→ `renderChildren()`（369 行）遍历 children |
| TabsWidget | `widgets/TabsWidget/widget/index.tsx:597-637` | `renderComponent()` 从 children 里过滤出当前选中 tab 的 canvas 调 `renderAppsmithCanvas`（636 行）—— 每个 tab 一个 canvas 子 widget |
| FormWidget | `widgets/FormWidget/widget/index.tsx:39` | 直接 `extends ContainerWidget`；因需感知子字段状态走 `childWidgets` 路径：`widgets/withWidgetProps.tsx:54` 的 `WIDGETS_WITH_CHILD_WIDGETS = ["LIST_WIDGET", "FORM_WIDGET"]` 经 `getChildWidgets` selector 注入；子树由 `utils/widgetRenderUtils.tsx:135` `buildChildWidgetTree()` 递归构建 |
| CANVAS_WIDGET | `widgets/CanvasWidget.tsx:13` | `class CanvasWidget extends ContainerWidget`，只是改 type/defaults（`detachFromLayout: true`） |
| Meta widget | `widgets/MetaWidgetContextProvider.tsx`、`MetaHOC.tsx`、`reducers/entityReducers/metaWidgetsReducer` | List/JSONForm/Tabs 动态生成子 widget；BaseWidget 提供 `modifyMetaWidgets` 等入口 |

### 5.2 布局系统如何参与渲染

`LayoutSystemTypes = FIXED | AUTO | ANVIL`（`layoutSystems/types/index.ts:6`）。每个布局系统导出 `get<X>LayoutSystem(renderMode)`，返回：

```ts
// layoutSystems/types/index.ts:28-64
LayoutSystem = {
  widgetSystem: { WidgetWrapper, propertyEnhancer },   // 包住每个 widget
  canvasSystem: { Canvas, propertyEnhancer },          // 渲染一层 canvas
}
```

| 布局系统 | 入口 | 特点 |
|---|---|---|
| fixed（绝对定位网格） | `layoutSystems/fixedlayout/index.ts:118` | editor 侧包裹层 `FixedLayoutEditorWidgetOnion.tsx:33`（AutoHeightOverlayLayer > PositionedComponentLayer > TabOrderBadge > Snipeable > Draggable > WidgetNameLayer > FixedResizableLayer > FixedLayoutWidgetComponent，层层"洋葱"）；viewer 侧 `FixedLayoutViewerWidgetOnion.tsx`；canvas 为 `FixedLayoutEditorCanvas.tsx` / `FixedLayoutViewerCanvas.tsx` |
| autolayout（旧 flex 模式，"animated-onion"） | `layoutSystems/autolayout/` | `AutoLayoutEditorWidgetOnion.tsx` / `AutoLayoutWidgetComponent.tsx` |
| anvil（新一代） | `layoutSystems/anvil/index.ts:54` | canvas 为 `AnvilEditorCanvas` / `AnvilViewerCanvas.tsx:34`（内部 `LayoutProvider` + `AnvilDetachedWidgets`）；**Anvil 的子组件排布不在 widget 内，而在 layout DSL**：`anvil/layoutComponents/LayoutProvider.tsx:8` 把 children 转成 ChildrenMapContext，`renderLayouts`（`anvil/utils/layouts/renderUtils.tsx`）按 zone/section 布局组件渲染，单个子 widget 由 `anvil/layoutComponents/WidgetRenderer.tsx:22` 渲染。Zone/Section 本身也是 widget（`widgets/wds/WDSZoneWidget/widget/index.tsx:39`） |

Widget 侧接入点：`withLayoutSystemWidgetHOC.tsx:36-57`（`LayoutSystemWrapper`：按 renderMode+layoutSystemType 取 widgetSystem，`propertyEnhancer(props)` 后 `<WidgetWrapper><Widget/></WidgetWrapper>`）。
Canvas 侧接入点：`layoutSystems/CanvasFactory.tsx:50` `renderAppsmithCanvas()`。

---

## 6. 渲染链路（编辑器画布 vs View 模式）

数据形态：服务端 DSL 树 → `INIT_CANVAS_LAYOUT/UPDATE_LAYOUT` 时 `denormalize("0", widgets)`（`ce/reducers/entityReducers/canvasWidgetsStructureReducer.ts:39`）生成树形 `canvasWidgetsStructure`；同时扁平 map 存在 `state.entities.canvasWidgets`（`sagas/selectors.tsx:75` `getWidget`）。

**编辑器**：

1. `pages/Editor/WidgetsEditor/components/MainContainerWrapper.tsx:121` 取 `getCanvasWidgetsStructure` → 传给 `pages/Editor/Canvas.tsx:105` 调 `renderAppsmithCanvas(widgetsStructure)`（Anvil 时外层套 `WDSThemeProvider`）。
2. `renderAppsmithCanvas`（`layoutSystems/CanvasFactory.tsx:44-52`）= `withWidgetProps(LayoutSystemBasedCanvas)`；按 renderMode(=CANVAS)+layoutSystemType 选 Canvas 实现，并做 canvas 级 propertyEnhancer。
3. Canvas（如 `FixedLayoutEditorCanvas`）对每个 child 调 `renderChildren`（`layoutSystems/common/utils/canvasUtils.ts:69`）→ `renderChildWidget`（22 行）→ **`WidgetFactory.createWidget(childWidget, renderMode)`（51 行）** → builder 渲染 HOC 链包装的 widget。
4. HOC 链内：`withWidgetProps.tsx:203-205` 把 `canvasWidget`（DSL）与 `evaluatedWidget`（求值结果）用 `createCanvasWidget()`（`utils/widgetRenderUtils.tsx:25`）合并成最终 props（求值未到时 `createLoadingWidget` 变成 SKELETON_WIDGET，92 行）；最终 `BaseWidget.render() → getWidgetView()`。
5. 容器 widget 的 `getWidgetView` 再 `renderAppsmithCanvas(child)` 递归。

**View 模式**：`pages/AppViewer/AppPage/AppPage.tsx:69` 同样 `renderAppsmithCanvas(widgetsStructure)`，**区别只在 renderMode=PAGE 时各布局系统选 Viewer 版 Canvas/Wrapper**。Modal 等 detached widget 由各布局系统的 `*DetachedWidgetOnion` / `AnvilDetachedWidgets` 单独渲染。

---

## 7. 以 ButtonWidget 为例：定义一个 widget 的全部文件

**传统（Fixed/AutoLayout）风格 —— `widgets/ButtonWidget/`：**

| 文件 | 导出 | 职责 |
|---|---|---|
| `index.ts` | `default`（re-export `./widget`） | 供 `widgets/index.ts` 动态 import |
| `widget/index.tsx:39` | `class ButtonWidget extends BaseWidget<ButtonWidgetProps, ButtonWidgetState>` + `default` | 全部静态元配置 + 事件处理 + `getWidgetView()`（661 行渲染 `ButtonComponent`） |
| `widget/index.tsx:702/724` | `ButtonWidgetProps extends WidgetProps` / `ButtonWidgetState` | widget 专属 props/state 类型 |
| `component/index.tsx` | `ButtonComponent`（+ `ButtonType`） | 纯 UI 渲染，不含业务逻辑 |
| `component/DragContainer.tsx`、`component/utils.tsx` | 辅助组件 | 拖拽容器等 |
| `icon.svg` / `thumbnail.svg` | — | `getConfig().iconSVG/thumbnailSVG` 引用（widget/index.tsx:30-31） |

ButtonWidget 类内静态成员清单：`type`(51)、`getConfig`(53)、`getDefaults`(64)、`getMethods`(85)、`getAutoLayoutConfig`(101)、`getAnvilConfig`(129)、`getAutocompleteDefinitions`(141)、`getPropertyPaneContentConfig`(153)、`getPropertyPaneStyleConfig`(297)、`getStylesheetConfig`(566)、`getMetaPropertiesMap`(576)、`getDerivedPropertiesMap`(582，返回 `{}`)、`getSetterConfig`(638)；实例方法 `onButtonClick/clickWithRecaptcha/getWidgetView` 等（586-699）。

**带派生属性的对照示例 —— `widgets/InputWidgetV2/widget/`：**

- `index.tsx:642` `getDerivedPropertiesMap()` 返回 `{ isValid: "{{(() => {${derivedProperties.isValid}})()}}" }`（把 JS 函数字符串塞进绑定表达式）
- `derived.js:2` —— 源码形态的派生函数（`(props, moment, _) => ...`）
- `parsedDerivedProperties.ts` —— 编译后的字符串版本（被 import）
- `Utilities.ts` / `Utilities.test.ts` —— 纯函数工具 + 测试
- `index.tsx:650` `getMetaPropertiesMap()` 返回 `{ inputText: "", text: "" }`；`:659` `getDefaultPropertiesMap()` 返回 `{ inputText: "defaultText", text: "defaultText" }`
- 继承：`class InputWidgetV2 extends BaseInputWidget`（`widgets/BaseInputWidget/widget/index.tsx:27`）

**Anvil/WDS 风格 —— `widgets/wds/WDSButtonWidget/`（config 拆文件）：**

- `widget/index.tsx:14` `class WDSButtonWidget extends BaseWidget<...>`，所有静态 getter 一行转发到 `../config`
- `widget/types.ts` —— Props/State 类型
- `component/`（index.tsx、Container.tsx、RecaptchaV2/V3.tsx、useRecaptcha.tsx）—— UI 层（基于 `@appsmith/ads`/wds）
- `config/`：`metaConfig.ts`、`defaultsConfig.ts`、`anvilConfig.ts`、`autocompleteConfig.ts`、`methodsConfig.ts`、`settersConfig.ts`、`propertyPaneConfig/{contentConfig,styleConfig,index}.ts`、`config/index.ts` 汇出

---

## 8. 「定义一个新 Widget 的清单」

1. `widgets/<Name>Widget/index.ts` —— `export default Widget`（re-export）。**入口 barrel**。
2. `widgets/<Name>Widget/widget/index.tsx` —— `class XxxWidget extends BaseWidget<XxxWidgetProps, XxxWidgetState>`，导出 `default`、`XxxWidgetProps`、`XxxWidgetState`。**widget 的"控制器"——元配置 + 事件 + getWidgetView**。
   - 必写静态成员：`static type = "XXX_WIDGET"`；`getConfig()`（名称/图标/tags/isCanvas/needsMeta/eagerRender/hideCard）；`getDefaults()`（rows/columns/widgetName/version 等）；`getPropertyPaneContentConfig()` + `getPropertyPaneStyleConfig()`；`getMetaPropertiesMap()`（有可变状态时）；`getDerivedPropertiesMap()`（有计算属性时）；`getDefaultPropertiesMap()`（meta 属性的 reset 来源）；`getSetterConfig()`；`getStylesheetConfig()`；`getAutocompleteDefinitions()`；`getAutoLayoutConfig()` / `getAnvilConfig()`；可选 `getMethods()`、`getFeatures()`、`getDependencyMap()`、`getLoadingProperties()`、`pasteOperationChecks()`。
   - 必写实例方法：`getWidgetView()`（渲染 component）；事件里用 `this.executeAction` / `this.updateWidgetMetaProperty`。
3. `widgets/<Name>Widget/component/index.tsx` —— 纯 UI 组件（受控、无 redux、无 DSL 访问）。**表现层**。
4. `widgets/<Name>Widget/icon.svg` + `thumbnail.svg` —— 面板图标与缩略图。
5. 派生属性（可选）：`widget/derived.js`（函数源码）+ `widget/parsedDerivedProperties.ts`（编译产物）+ `widget/Utilities.ts`。
6. 容器型 widget 额外：`getDefaults().blueprint`（自动建子 CANVAS_WIDGET）+ `getConfig().isCanvas: true`，`getWidgetView` 内用 `renderAppsmithCanvas(childWidget)` 递归（参照 `ContainerWidget/widget/index.tsx:345-366`）。
7. **注册点（必改）**：`widgets/index.ts` 加 loader entry `[ "XXX_WIDGET", async () => import("./XxxWidget").then(m => m.default) ]`（CE widget；EE widget 走 `ee/widgets` 的 `EEWidgets`）。
8. Anvil 新式写法：`widget/config/*` 拆分 + `component/` 基于 wds；参照 `widgets/wds/WDSButtonWidget/`。
9. 可选周边：`WidgetQueryGenerators/`（一键绑定查询生成，经 `getMethods().getQueryGenerationConfig`）、`utils/WidgetFeatures.ts` 注册 feature、测试 `widget/index.test.tsx` / `component/index.test.tsx`。

---

## 9. 关键类型/类关系图（文字版）

```
WidgetType (string, = widget.type)
   │
   ├── widgets/index.ts  WidgetLoaders: Map<type, () => Promise<typeof BaseWidget>>
   │        │ (import 副作用 registerWidgetLoaders)
   │        ▼
   │   widgets/registry.ts  loadWidget / loadAllWidgets  ──(editorInitializer / EvaluationsSaga)──▶
   │        │ registerWidgets(widgetClasses)
   │        ▼
   │   WidgetProvider/factory/registrationHelper.tsx: registerWidget
   │        │ withBaseWidgetHOC(widget, needsMeta, eagerRender) → builder
   │        ▼
   │   WidgetFactory (widgetsMap / widgetTypes / widgetBuilderMap / widgetConfigMap ...)
   │        │ createWidget(dslNode, renderMode) ──▶ builder(props) ──▶ HOC 链
   │        ▼
   │   HOC 链: withWidgetProps → withLazyRender → withLayoutSystemWidgetHOC → [withMeta] → BaseWidget 子类
   │
   └── BaseWidget<T = WidgetProps, K = WidgetState> (abstract, React.Component)
            │ 静态元信息（defaults/config/propertyPane*/derived/default/meta/...）
            │   ── 被 WidgetFactory.configureWidget 摊平进 widgetConfigMap（组件面板卡片、加 widget 入 DSL）
            │   ── getWidgetTypeConfigMap() 供求值 worker：defaultProperties / derivedProperties / metaProperties
            │ 实例: executeAction / updateWidget* / modifyMetaWidgets / getWidgetView()
            ▼
        WidgetProps（BaseWidget.tsx:570）
            = WidgetBaseProps + WidgetPositionProps + WidgetDisplayProps + ... + [key:string]:any
            运行时来源（withWidgetProps.tsx:203）:
              FlattenedWidgetProps(DSL/Redux) ⊕ WidgetEntity(求值结果) ⊕ meta ⊕ layoutSystem props
```

**渲染链路一句话总结**：DSL 树（canvasWidgetsStructure）→ `renderAppsmithCanvas`（CanvasFactory:50）→ 当前布局系统 Canvas → `renderChildWidget`（canvasUtils:22）→ `WidgetFactory.createWidget`（factory/index.tsx:187）→ HOC 链（取数 + 懒渲染 + 布局洋葱 + meta）→ `BaseWidget.getWidgetView()` → 纯 UI `component/`。编辑器与 view 模式的差别仅在 renderMode（CANVAS vs PAGE）导致布局系统选择 Editor/Viewer 两套 Wrapper 与 Canvas。

---

## 10. 设计要点小结（可借鉴的抽象）

1. **逻辑/表现两层分离**：`widget/`（控制器：元配置+事件+DSL 写入）与 `component/`（受控 UI）分离，且用基类约定禁止 component 触碰 Redux/DSL —— UI 可独立测试与复用。
2. **静态 getter 即元数据**：一个类的静态方法集合 = widget 的全部元配置（默认值、面板、派生属性、setter、自动补全），工厂统一摊平消费；属性面板/求值引擎/组件面板都从这一个来源取数，**单一事实源**。
3. **注册表 + 动态 import**：widget 按需懒加载（worker 端甚至只加载 DSL 用到的类型），新增 widget 零侵入（只加 loader entry）。
4. **HOC 洋葱层组合横切关注**：取数、懒渲染、布局包装、meta 注入各自独立成 HOC，按需叠加。
5. **布局系统可插拔**：fixed/autolayout/anvil 三套布局通过 `LayoutSystem` 接口（widgetSystem + canvasSystem）接入同一渲染管线，widget 代码不感知具体布局。
6. **蓝图（blueprint）机制**：容器拖入即自动生成子结构（canvas/tab），声明式描述"这个 widget 生来长什么样"。
