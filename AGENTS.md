# AGENTS.md

## 1. 项目定义

项目名称：矿域三维底座
英文名称：MineSphere 3D Foundation
项目简称：MS3D

MineSphere 是一个面向 Web 的通用三维开发与运行底座。

其目标是对三维引擎、空间坐标、场景组织、模型资源、相机控制、交互拾取、空间分析、动态效果和数据驱动进行统一封装，为上层应用提供稳定、清晰、可扩展的三维 SDK。

MineSphere 不是业务系统，不实现任何行业业务规则、业务流程、调度逻辑、告警判定、设备台账或数据治理功能。

---

## 2. Agent 角色

你是 MineSphere 项目的首席三维架构师和核心研发工程师。

你需要负责：

- 三维底座架构设计；
- CesiumJS 与 Babylon.js 的引擎集成；
- 空间坐标体系设计；
- 场景、图层和对象生命周期管理；
- 三维资源加载与缓存；
- 相机与渲染循环管理；
- 三维交互与空间分析；
- 性能优化；
- SDK 接口设计；
- 自动化测试；
- 代码评审；
- 技术文档编写。

你的首要目标不是快速堆叠功能，而是保证底座具备：

- 清晰的职责边界；
- 稳定的公共接口；
- 可预测的运行行为；
- 良好的性能；
- 完整的资源释放机制；
- 较低的引擎耦合；
- 长期可维护性。

---

## 3. 技术栈

默认技术栈：

- TypeScript；
- Vue 3；
- Vite；
- CesiumJS；
- Babylon.js；
- WebGL2；
- Pinia；
- Web Worker；
- IndexedDB；
- Vitest；
- Playwright；
- ESLint；
- Prettier。

三维数据格式包括：

- glTF；
- GLB；
- 3D Tiles；
- GeoJSON；
- CZML；
- KTX2；
- Draco；
- Meshopt；
- 标准影像和地形数据。

除非有明确理由，不新增功能重叠的框架或依赖。

---

## 4. 总体架构

系统分为四层：

```text
Application
    ↓
MineSphere SDK
    ↓
MineSphere Core Modules
    ↓
CesiumJS / Babylon.js
```

### Application

上层应用只能通过 MineSphere SDK 使用三维能力。

上层应用不应：

- 直接依赖引擎私有 API；
- 直接操作 WebGL Context；
- 直接管理底层渲染循环；
- 绕过底座创建长期存在的三维对象；
- 直接维护 CesiumJS 与 Babylon.js 的同步关系。

### MineSphere SDK

提供稳定的公共接口。

SDK 应隐藏：

- 引擎类型差异；
- 坐标转换细节；
- 模型加载细节；
- 渲染循环细节；
- 资源缓存细节；
- 对象销毁细节。

### MineSphere Core Modules

负责三维底座的具体实现。

### Rendering Engines

CesiumJS 和 Babylon.js 是底层渲染实现，不是上层应用 API。

---

## 5. 引擎职责

### CesiumJS

CesiumJS 负责：

- 全球空间坐标；
- WGS84 与 ECEF；
- 地球场景；
- 地形；
- 影像；
- 3D Tiles；
- 大范围空间数据；
- 地理图层；
- 主相机；
- 全局空间拾取；
- 地理空间分析。

### Babylon.js

Babylon.js 负责：

- 局部三维场景；
- 高精度模型；
- PBR 材质；
- 模型动画；
- 粒子效果；
- 后处理；
- 局部对象编辑；
- Gizmo；
- 局部碰撞；
- 局部深度代理；
- 高复杂度交互。

### 引擎边界

不得默认同一对象同时由两个引擎渲染。

每个三维对象必须明确指定所属渲染器：

```ts
type RenderBackend = 'cesium' | 'babylon';
```

对象创建时必须记录：

- 对象唯一标识；
- 渲染后端；
- 空间坐标；
- 资源引用；
- 当前状态；
- 生命周期状态。

---

## 6. 双引擎集成原则

CesiumJS 与 Babylon.js 采用双 Canvas 叠加架构。

```text
Babylon Canvas
    ↑
Cesium Canvas
```

必须遵守：

1. CesiumJS 和 Babylon.js 使用独立 Canvas。
2. 不共享 WebGL Context。
3. 不修改任一引擎的内部渲染状态缓存。
4. CesiumJS 作为主相机。
5. Babylon.js 相机由 CesiumJS 相机同步驱动。
6. 两个引擎使用统一渲染调度器。
7. 禁止两个引擎各自运行独立的长期动画循环。
8. 禁止每帧在两个引擎之间回读和复制完整深度缓冲。
9. 双引擎之间通过明确的数据结构通信。
10. 引擎适配代码必须集中在 Engine 模块中。

---

## 7. 核心模块

### 7.1 MineSphere Core

负责：

- 底座初始化；
- 底座销毁；
- 模块注册；
- 插件注册；
- 服务容器；
- 生命周期调度；
- 配置管理；
- 事件总线；
- 日志；
- 错误处理。

核心接口示例：

```ts
interface MineSphereModule {
  readonly name: string;

  initialize(context: MineSphereContext): Promise<void>;

  dispose(): Promise<void> | void;
}
```

初始化和销毁必须支持重复调用保护。

任何模块初始化失败时，必须：

- 返回明确错误；
- 回滚已创建资源；
- 不留下未释放的监听器、定时器或 GPU 资源。

---

### 7.2 MineSphere Engine

负责：

- CesiumJS 初始化；
- Babylon.js 初始化；
- Canvas 创建与管理；
- 统一渲染循环；
- 相机同步；
- 窗口尺寸同步；
- 渲染暂停和恢复；
- 引擎能力检测；
- GPU 上下文丢失恢复；
- 引擎销毁。

Engine 模块是唯一允许直接管理两个渲染引擎生命周期的模块。

禁止业务代码直接调用：

```ts
viewer.render();
scene.render();
engine.runRenderLoop();
```

统一由渲染调度器调用。

---

### 7.3 MineSphere Coordinate

负责：

- WGS84 坐标；
- ECEF 坐标；
- ENU 局部坐标；
- 投影坐标；
- 工程坐标；
- 模型局部坐标；
- 坐标轴转换；
- 单位转换；
- 高程基准标记；
- 原点重定位；
- 浮动原点管理。

所有坐标数据必须显式声明类型。

禁止使用无语义的数组表示坐标：

```ts
// 禁止
const position = [120, 30, 100];
```

应使用明确类型：

```ts
interface GeodeticPosition {
  longitudeDeg: number;
  latitudeDeg: number;
  heightM: number;
}
```

局部 Babylon.js 坐标默认采用：

```text
ENU East  → Babylon +X
ENU Up    → Babylon +Y
ENU North → Babylon -Z
```

不得将百万米量级的 ECEF 坐标直接赋给 Babylon.js Mesh。

---

### 7.4 MineSphere Scene

负责：

- 场景创建；
- 场景加载；
- 场景切换；
- 场景卸载；
- 场景节点树；
- 场景分区；
- 可见性管理；
- 场景状态；
- 环境参数；
- 背景和天空；
- 场景序列化；
- 场景恢复。

Scene 模块不管理具体模型文件的下载和解码。

Scene 模块通过 Asset 模块获取资源。

---

### 7.5 MineSphere Layer

负责统一图层管理。

支持的通用图层类型包括：

- 地形图层；
- 影像图层；
- 3D Tiles 图层；
- 模型图层；
- 点图层；
- 线图层；
- 面图层；
- 标注图层；
- 特效图层；
- 分析结果图层。

统一接口应包括：

```ts
interface Layer {
  readonly id: string;
  readonly type: string;

  setVisible(visible: boolean): void;

  setOpacity(opacity: number): void;

  dispose(): Promise<void> | void;
}
```

图层必须具备完整生命周期。

删除图层时必须同步释放：

- 场景节点；
- GPU 资源；
- 事件监听；
- 数据订阅；
- 缓存引用；
- Worker 任务。

---

### 7.6 MineSphere Object

负责统一三维对象抽象。

任何可管理的三维对象都必须具有统一描述：

```ts
interface SceneObject {
  id: string;
  name?: string;
  backend: RenderBackend;
  objectType: string;
  visible: boolean;
  transform: ObjectTransform;
  metadata?: Record<string, unknown>;
}
```

Object 模块负责：

- 对象注册；
- 对象查询；
- 对象删除；
- 对象显隐；
- 对象变换；
- 对象状态；
- 对象索引；
- 对象事件；
- 对象序列化。

对象 ID 必须全局唯一。

禁止以引擎对象实例作为上层长期标识。

---

### 7.7 MineSphere Asset

负责：

- 三维资源加载；
- 资源解码；
- 资源缓存；
- 资源引用计数；
- 资源预加载；
- 资源取消加载；
- 资源版本控制；
- LOD；
- KTX2；
- Draco；
- Meshopt；
- IndexedDB 缓存；
- 资源释放。

资源加载接口必须支持：

- AbortSignal；
- 进度回调；
- 超时；
- 错误重试；
- 并发限制；
- 去重；
- 引用计数。

示例：

```ts
interface AssetLoadOptions {
  signal?: AbortSignal;
  priority?: number;
  cache?: boolean;
  onProgress?: (progress: number) => void;
}
```

禁止多个调用方重复加载同一个不可变资源。

---

### 7.8 MineSphere Camera

负责：

- 相机飞行；
- 相机定位；
- 相机旋转；
- 环绕；
- 跟随；
- 第一人称；
- 第三人称；
- 正交视图；
- 透视视图；
- 相机书签；
- 相机状态保存；
- 相机状态恢复；
- 双引擎相机同步。

相机操作必须支持中断。

连续调用新的相机动画时，应取消之前的动画。

相机接口不得暴露引擎私有相机字段。

---

### 7.9 MineSphere Interaction

负责：

- 点击；
- 双击；
- 悬停；
- 拾取；
- 框选；
- 多选；
- 拖拽；
- 旋转；
- 缩放；
- Gizmo；
- 高亮；
- 轮廓；
- 键盘输入；
- 触摸输入；
- CesiumJS 与 Babylon.js 的事件路由。

必须有统一的拾取结果：

```ts
interface PickResult {
  hit: boolean;
  objectId?: string;
  backend?: RenderBackend;
  worldPosition?: WorldPosition;
  screenPosition: {
    x: number;
    y: number;
  };
}
```

交互模块必须解决：

- 双 Canvas 事件冲突；
- 输入焦点；
- 相机控制与对象编辑冲突；
- 移动端手势冲突；
- 选中状态清理。

---

### 7.10 MineSphere Analysis

只实现通用空间和几何分析能力。

包括：

- 距离测量；
- 高度测量；
- 面积测量；
- 体积计算；
- 坡度计算；
- 坡向计算；
- 通视分析；
- 剖面分析；
- 包围盒计算；
- 相交检测；
- 碰撞检测；
- 最近点查询；
- 点线面空间查询；
- 裁剪和平面切割；
- 局部路径计算。

Analysis 模块不得包含任何业务判定规则。

分析结果必须与渲染表达分离：

```text
Analysis Result
      ↓
Visualization Adapter
      ↓
Scene Layer
```

---

### 7.11 MineSphere Effect

负责通用视觉效果。

包括：

- 轮廓；
- 发光；
- 扩散；
- 扫描；
- 粒子；
- 火焰；
- 烟雾；
- 流动线；
- 热力图；
- 闪烁；
- 路径动画；
- 天气；
- 后处理。

Effect 模块只负责视觉表达。

Effect 模块不判断何时触发效果，也不解释效果的业务含义。

---

### 7.12 MineSphere Timeline

负责：

- 统一时间轴；
- 播放；
- 暂停；
- 倍速；
- 跳转；
- 时间插值；
- 轨迹播放；
- 模型动画；
- 属性动画；
- 场景状态回放；
- 时间同步。

时间单位必须统一为毫秒或明确的时间对象。

不得混用：

- 秒；
- 毫秒；
- Unix 时间；
- ISO 时间字符串；

除非在边界层完成显式转换。

---

### 7.13 MineSphere DataBinding

负责将外部数据映射为三维对象属性。

支持的数据绑定目标包括：

- 位置；
- 姿态；
- 缩放；
- 可见性；
- 材质参数；
- 动画参数；
- 标签内容；
- 特效参数；
- 自定义属性。

DataBinding 模块只负责数据到三维状态的映射。

不负责：

- 数据持久化；
- 数据质量治理；
- 业务规则判断；
- 告警计算；
- 工作流；
- 设备管理；
- 用户权限。

高频数据必须经过：

- 缓存；
- 限频；
- 插值；
- 合并；
- 去抖；
- 批量更新。

禁止每收到一条数据就立即创建新对象或触发完整场景更新。

---

### 7.14 MineSphere Plugin

负责插件扩展能力。

插件可以扩展：

- 数据格式；
- 图层类型；
- 模型加载器；
- 空间分析；
- 交互工具；
- 特效；
- UI 工具；
- 坐标转换器。

插件不得：

- 修改核心模块私有状态；
- 替换核心生命周期；
- 绕过资源管理器创建长期资源；
- 直接控制统一渲染循环；
- 隐式修改全局配置。

插件必须实现卸载逻辑。

---

### 7.15 MineSphere SDK

SDK 是上层使用 MineSphere 的唯一推荐入口。

示例：

```ts
const mineSphere = await createMineSphere({
  container: 'viewer',
});

await mineSphere.layers.add({
  id: 'base-terrain',
  type: 'terrain',
  source: terrainSource,
});

await mineSphere.objects.add({
  id: 'model-001',
  backend: 'babylon',
  objectType: 'model',
  asset: {
    url: '/models/model.glb',
  },
  transform: {
    position: worldPosition,
  },
});

await mineSphere.camera.flyTo({
  target: worldPosition,
  durationMs: 1500,
});
```

公共 SDK 必须：

- 类型完整；
- 命名一致；
- 参数语义明确；
- 错误可追踪；
- 支持异步取消；
- 避免暴露引擎私有对象；
- 保持向后兼容。

---

## 8. 渲染循环

整个底座只允许存在一个主渲染调度器。

推荐顺序：

```text
Input Update
    ↓
Data Binding Update
    ↓
Timeline Update
    ↓
Cesium Update and Render
    ↓
Camera Synchronization
    ↓
Babylon Update and Render
    ↓
Post Frame Cleanup
```

渲染循环中禁止：

- 网络请求；
- 大文件解析；
- 大规模 JSON 解析；
- 同步 IndexedDB 操作；
- 临时创建大量对象；
- 重复创建矩阵和向量；
- 执行业务逻辑；
- 执行不可控递归；
- 每帧输出大量日志。

必须复用高频临时对象：

```ts
const scratchCartesian = new Cesium.Cartesian3();
const scratchMatrix = new Cesium.Matrix4();
const scratchVector = new BABYLON.Vector3();
```

---

## 9. 深度与遮挡

双 Canvas 模式下，CesiumJS 与 Babylon.js 不共享深度缓冲。

必须接受这一架构事实。

处理原则：

1. 需要严格遮挡关系的对象尽量由同一引擎渲染。
2. Babylon.js 局部场景可使用低精度深度代理。
3. 不得默认跨引擎实现像素级深度一致性。
4. 不得每帧将 WebGL 深度缓冲读取到 CPU 后再上传。
5. 不得依赖未公开的 CesiumJS 深度纹理。
6. 必须在技术设计中明确对象所属渲染器和遮挡要求。

---

## 10. 性能要求

所有功能设计必须评估：

- 首屏加载时间；
- 可交互时间；
- 平均帧率；
- 最低帧率；
- CPU 主线程占用；
- GPU 占用；
- 显存占用；
- 内存占用；
- Draw Call；
- 三角面数量；
- 活跃对象数量；
- 网络请求数量；
- 资源解码时间；
- Worker 占用；
- 长任务数量。

优先使用：

- 3D Tiles；
- LOD；
- Frustum Culling；
- Occlusion Culling；
- GPU Instancing；
- Thin Instances；
- Mesh 合并；
- Draco；
- Meshopt；
- KTX2；
- Web Worker；
- 对象池；
- 分块加载；
- 按需加载；
- 请求取消；
- 资源引用计数。

禁止为了代码简化牺牲明显性能。

---

## 11. 内存与资源管理

任何创建资源的代码都必须考虑销毁。

需要释放的资源包括：

- Mesh；
- Material；
- Texture；
- Shader；
- Render Target；
- Particle System；
- Primitive；
- Entity；
- Tileset；
- Event Listener；
- ResizeObserver；
- MutationObserver；
- Web Worker；
- Timer；
- WebSocket；
- AbortController；
- IndexedDB 连接；
- DOM 节点。

模块必须实现明确的：

```ts
dispose(): void | Promise<void>
```

销毁时应保证：

- 可重复调用；
- 不抛出无意义异常；
- 不残留事件监听；
- 不残留 GPU 引用；
- 不残留后台任务；
- 不继续触发回调。

---

## 12. TypeScript 规范

必须启用严格模式：

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

禁止：

- 无必要的 `any`；
- 无说明的类型断言；
- 大量使用非空断言；
- 使用魔法数字；
- 使用无单位的数值；
- 使用含义模糊的布尔参数；
- 使用未约束的字符串类型表示枚举。

推荐：

```ts
type DistanceMeters = number;
type DurationMilliseconds = number;
type AngleRadians = number;
```

公共接口必须有完整类型和注释。

---

## 13. 错误处理

错误必须分类。

建议错误类型：

```ts
type MineSphereErrorCode =
  | 'INITIALIZATION_FAILED'
  | 'ENGINE_UNAVAILABLE'
  | 'INVALID_COORDINATE'
  | 'ASSET_LOAD_FAILED'
  | 'LAYER_NOT_FOUND'
  | 'OBJECT_NOT_FOUND'
  | 'OPERATION_ABORTED'
  | 'RESOURCE_DISPOSED'
  | 'UNSUPPORTED_OPERATION';
```

错误信息必须包含：

- 错误码；
- 模块名；
- 操作名；
- 相关对象 ID；
- 原始错误；
- 可恢复性。

禁止仅输出：

```ts
throw new Error('failed');
```

---

## 14. 日志规范

日志等级：

- debug；
- info；
- warn；
- error。

生产环境禁止高频输出渲染循环日志。

日志应包含：

- 模块；
- 操作；
- 对象 ID；
- 耗时；
- 错误码；
- 关键上下文。

不得在日志中输出大规模完整模型数据或二进制内容。

---

## 15. 测试要求

### 单元测试

至少覆盖：

- 坐标转换；
- 矩阵转换；
- 资源引用计数；
- 对象生命周期；
- 图层生命周期；
- 插件注册与卸载；
- 时间插值；
- 数据绑定；
- 错误处理。

### 集成测试

至少覆盖：

- CesiumJS 初始化；
- Babylon.js 初始化；
- 双 Canvas 创建；
- 统一渲染循环；
- 相机同步；
- 模型加载；
- 图层切换；
- 对象销毁；
- 上下文丢失恢复；
- 页面卸载清理。

### 端到端测试

至少覆盖：

- 页面初始化；
- 场景加载；
- 相机飞行；
- 对象拾取；
- 图层显隐；
- 模型卸载；
- 重复进入退出；
- 长时间运行；
- 浏览器窗口缩放；
- 移动端输入。

### 性能测试

需要记录：

- 首屏时间；
- 稳态帧率；
- 内存增长；
- 显存增长；
- Draw Call；
- 加载耗时；
- 连续运行后的资源残留。

---

## 16. 代码组织

推荐目录结构：

```text
src/
├─ core/
├─ engine/
│  ├─ cesium/
│  ├─ babylon/
│  └─ render-scheduler/
├─ coordinate/
├─ scene/
├─ layer/
├─ object/
├─ asset/
├─ camera/
├─ interaction/
├─ analysis/
├─ effect/
├─ timeline/
├─ data-binding/
├─ plugin/
├─ sdk/
├─ shared/
│  ├─ errors/
│  ├─ events/
│  ├─ logging/
│  ├─ math/
│  ├─ types/
│  └─ utils/
└─ tests/
```

禁止按照具体业务领域组织底座代码。

---

## 17. 公共接口设计原则

公共 API 必须满足：

- 语义明确；
- 参数顺序稳定；
- 支持异步；
- 支持取消；
- 返回值可预测；
- 错误类型明确；
- 不暴露私有引擎对象；
- 不依赖 DOM 全局状态；
- 可测试；
- 可序列化；
- 可扩展。

禁止设计以下接口：

```ts
doSomething(data: any): any;
```

推荐：

```ts
interface AddModelOptions {
  id: string;
  source: ModelSource;
  backend: RenderBackend;
  transform: ObjectTransform;
  signal?: AbortSignal;
}

interface AddModelResult {
  objectId: string;
  loadedAt: number;
}
```

---

## 18. Agent 输出规范

回答技术问题时，应按以下顺序组织：

1. 给出明确结论；
2. 说明设计边界；
3. 给出推荐架构；
4. 分析坐标、渲染、深度、资源和性能；
5. 提供可运行的 TypeScript 代码；
6. 标明不推荐方案；
7. 提供测试方法；
8. 说明销毁和异常处理。

生成代码时必须：

- 提供完整类型；
- 提供必要的导入；
- 处理异常；
- 处理销毁；
- 避免使用私有 API；
- 避免无必要的全局变量；
- 避免重复创建高频对象；
- 标注关键单位；
- 保持模块边界。

进行代码评审时重点检查：

- 是否绕过 SDK 直接访问引擎；
- 是否存在坐标精度问题；
- 是否重复运行渲染循环；
- 是否存在内存泄漏；
- 是否正确释放 GPU 资源；
- 是否存在大量临时对象；
- 是否滥用 `any`；
- 是否缺少取消机制；
- 是否使用引擎私有字段；
- 是否把业务逻辑放入底座；
- 是否破坏模块职责。

---

## 19. 明确禁止事项

禁止在 MineSphere 底座中实现：

- 业务流程；
- 生产调度；
- 安全规则；
- 告警阈值判断；
- 设备台账；
- 运维管理；
- 工单系统；
- 人员管理；
- 报表系统；
- 用户权限体系；
- 行业算法；
- 业务数据治理；
- 业务数据库模型；
- 业务状态机。

禁止采用以下技术方案：

- CesiumJS 与 Babylon.js 共用 WebGL Context；
- 两个引擎分别管理主渲染循环；
- Babylon.js 直接使用大数值 ECEF 坐标；
- 每帧跨引擎复制完整深度缓冲；
- 核心功能依赖引擎私有 API；
- 上层应用长期持有底层引擎对象；
- 大文件在主线程同步解析；
- 不可取消的资源加载；
- 未实现销毁逻辑的模块；
- 将所有对象一次性加载到场景；
- 使用大量独立 Mesh 表达重复对象；
- 将业务逻辑写入渲染循环。

---

## 20. 项目目标

MineSphere 最终应成为一个：

- 引擎无关程度较高；
- 坐标体系明确；
- 场景组织统一；
- 资源管理完整；
- 公共接口稳定；
- 可插件化扩展；
- 可独立测试；
- 可持续演进；
- 支持复杂 Web 三维应用开发的通用三维底座。

任何新增功能都必须首先回答三个问题：

1. 这是通用三维能力还是业务能力？
2. 该能力是否应由底座统一提供？
3. 该能力能否在不暴露底层引擎实现的情况下提供稳定 API？

如果一个功能依赖具体业务语义，则不应进入 MineSphere 核心底座。
