# Firefox Profiler 架构速览（面向 C++/UE4 工程师）

| 主题 | 说明 | 关键文件/代码 |
| ---- | ---- | ---- |
| 总览视角 | Firefox Profiler 分为 Gecko 侧的原始采集、浏览器内的数据传输，以及 profiler.firefox.com 上的 React/Redux 客户端三层：Gecko 负责采样、标记与符号信息收集；浏览器通过 JSON 把 profile 发给站点；前端再做大量离线处理与渲染。【F:docs-developer/architecture.md†L1-L26】【F:docs-developer/gecko-profile-format.md†L1-L62】 |
| &nbsp;&nbsp;&nbsp;&nbsp;核心子系统 | `src/profile-logic` 承担所有重计算与数据派生；Redux action/reducer/selector 只是组织数据流；React 组件负责可视化与交互，形成“C++ 后端 + 数据管线 + UI 外壳”的心智模型。【F:src/README.md†L1-L14】【F:src/profile-logic/README.md†L1-L16】 |
| &nbsp;&nbsp;&nbsp;&nbsp;应用引导 | 页面入口先创建 Redux store、挂载 `<Root>` 组件，并暴露 `connectToGeckoProfiler` Promise 给浏览器原生端；可以把它想成在主线程初始化引擎上下文并等待外部数据源接入。【F:src/index.js†L19-L107】【F:src/app-logic/create-store.js†L7-L35】 |
| React/Redux 心智模型（类比游戏引擎） | Redux `actions → reducers → selectors` 相当于“指令缓冲 → 状态同步 → 只读视图”。组件大多是 PureComponent，利用 memoized selector 避免不必要的 diff，可类比于引擎中按需刷新的渲染缓存；`connect()` 则像把派生数据绑定到 UI 层脚本。【F:src/components/README.md†L1-L39】【F:src/reducers/README.md†L1-L35】【F:src/selectors/README.md†L1-L16】 |
| &nbsp;&nbsp;&nbsp;&nbsp;app-logic 层 | 用来封装“场景管理”式的流程：存放跨视图的业务逻辑、会话控制和 store 工厂，避免在渲染层混入复杂过程代码。【F:src/app-logic/README.md†L1-L8】 |
| 性能数据管线 | 原始 Gecko JSON 以 `{schema,data}` 的表格数组传递，首先通过 `_toStructOfArrays` 转成列式数组以减少 JS 对象数，类似把 AoS 转成 SoA 提高缓存效率；随后 `GlobalDataCollector` 去重库与字符串，为多进程合并做准备。【F:docs-developer/gecko-profile-format.md†L63-L154】【F:src/profile-logic/process-profile.js†L108-L137】【F:src/profile-logic/process-profile.js†L182-L226】 |
| &nbsp;&nbsp;&nbsp;&nbsp;函数与资源提取 | `extractFuncsAndResourcesFromFrameLocations` 根据帧字符串解析出函数表与资源表，补齐 Gecko 原始数据缺失的函数元信息，相当于“离线符号解析 + 资源注册”步骤。【F:src/profile-logic/process-profile.js†L245-L313】 |
| &nbsp;&nbsp;&nbsp;&nbsp;派生表构建 | `profile-logic` 把栈、样本、分配等表转换成统一的 Struct-of-Arrays；`RawStackTable` 通过 prefix 指针压缩调用树，`RawSamplesTable` 支持采样/Tracing/字节权重，`JsAllocationsTable` 把 JS 分配转成可与采样共享的栈索引。【F:src/types/profile.js†L39-L137】【F:src/types/profile.js†L139-L155】 |
| 调用栈分析算法 | Call Tree 以函数而非帧聚合，计算 self/total time 并用 `CallNodePath` 唯一定位节点，避免同一函数多帧导致的树膨胀；理解上可类比于把原始栈样本压缩成“函数状态机”。【F:docs-developer/call-tree.md†L1-L162】 |
| 火焰图/Stack Chart | `getStackTimingByDepth` 逐样本维护“开放盒子”栈，重建每个调用区间的开始/结束时间，同时生成 same-width 索引，既能画真实宽度火焰图，也能绘制等宽视图；算法和引擎中的区间批处理类似，强调缓存和二分查找友好结构。【F:src/profile-logic/stack-timing.js†L13-L198】 |
| 时间线轨道调度 | `tracks.js` 固定轨道 ID 以保持 URL 兼容，并根据类型/活动度排序线程、IPC、功耗等轨道，类似根据优先级为调试 HUD 布局；还对本地/全局轨道使用独立排序规则。【F:src/profile-logic/tracks.js†L31-L200】 |
| 符号化流程 | 文档详细描述地址→库→偏移→符号的多级解析：先查 IndexedDB 缓存，再调用 Mozilla 符号化 API，最后回退到浏览器扩展执行 `dump_syms`/`nm`；理解上接近 C++ 构建流水线的增量符号加载。【F:docs-developer/symbolication.md†L1-L156】【F:docs-developer/symbolication.md†L157-L225】 |
| Gecko 交互接口 | Gecko Profiler 可通过 `Services.profiler` 独立启动、采集并返回 JSON，包含线程、样本、marker、库等表；浏览器端的 about:profiling 与 WebExtension API 直接调用这一 C++ 组件，Web 前端只需等待数据流入。【F:docs-developer/architecture.md†L20-L31】【F:docs-developer/gecko-profile-format.md†L1-L120】 |
| 关键数据结构索引 | `ProfileMeta`、线程表、帧/函数/栈表、样本表与分配表都在 Flow 类型中定义，是理解任何算法的权威入口，相当于头文件：可查每列含义及索引关系。【F:src/types/profile.js†L11-L155】 |
| 内存/分配追踪 | JS 分配在预处理阶段从 marker 抽离成 `JsAllocationsTable`，带有时间戳、Nursery 标记与栈索引，可与采样共用调用路径进行分配/释放配对分析。【F:src/types/profile.js†L139-L155】 |
| 进一步阅读指引 | 想要在 C++ 侧扩展/集成，可从 `docs-developer/processed-profile-format.md` 了解列式格式，再参考 `docs-developer/upgrading-profiles.md` 的升级策略以保持版本兼容。【F:docs-developer/processed-profile-format.md†L1-L13】【F:docs-developer/upgrading-profiles.md†L1-L78】 |
