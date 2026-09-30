# CNC 加工仿真系统 — Unity3D 集成说明

本工程分两层：

- `Assets/CncSim/Scripts/Core/` — 与引擎无关的功能内核（纯 C#，`noEngineReferences`），输出 `MeshData` / `Float3` / `ColorRgba`。
- `Assets/CncSim/Scripts/Runtime/` — Unity 表现层（MonoBehaviour、Mesh、材质、相机交互），把 Core 数据转成 Unity 对象。

UI 布局由你自行编写；Runtime 层只提供功能装配与事件接口。

## 目录与功能对照

| 目录/文件 | 覆盖功能 |
|---|---|
| `Core/CodeEditor/` | 1-8 编辑器、语法高亮、行号、文件、最近文件、导入导出、自动保存恢复 |
| `Core/Parsing/` | 9-17 解析、G/M 指令、坐标提取、运动队列、插补、错误检查、超程、单行解释 |
| `Core/Tooling/`、`Core/Stock/` | 18-30 刀具库、类型/直径/圆弧/长度/伸出/T 号、毛坯尺寸形状材料、G54-G59、夹具、对刀 |
| `Core/Kinematics/`、`Core/Parsing/MachineProfile.cs`、`Core/MachineModel/` | 31、32-36、81-83 机床类型、运动学、自定义机床、各部件几何 |
| `Runtime/Scene/SceneEntityRenderer.cs` | 32-36 主轴/工作台/床身/夹具/刀具/毛坯三维模型 |
| `Runtime/Rendering/ToolpathRenderer.cs` | 37-38 轨迹实时显示、线框/实体切换 |
| `Runtime/Camera/CameraRig.cs` | 39-40 多视角、缩放/旋转/平移 |
| `Runtime/Scene/TransparencyController.cs` | 41 透明度 |
| `Runtime/Scene/SectionView.cs` | 42 截面视图 |
| `Runtime/Interaction/MeasurementTool.cs` | 43 距离/角度测量 |
| `Runtime/Scene/SceneGrid.cs` | 44 坐标系/网格 |
| `Core/Simulation/`、`Runtime/CncSimBehaviour.cs` | 45-55 连续/单步/暂停/停止、速度、进度、指定行开始、材料去除、表面更新、切削动画 |
| `Core/Collision/`、`Core/Validation/`、`Core/Analysis/` | 56-61 碰撞、过切/欠切、限位 |
| `Core/Simulation/TimeEstimator.cs`、`CuttingCalculator.cs` | 63-67 时间/切削速度/进给/转速/参数推荐 |
| `Core/Parsing/CodeDatabase.cs` | 68-69 G/M 速查 |
| `Core/Optimization/` | 70 路径优化建议 |
| `Core/PostProcessing/` | 84 自定义后处理 |
| `Core/Performance/`、`Runtime/Rendering/RenderBatcher.cs` | 85-90 后台预计算、网格简化、批量合并、GPU 实例化、自适应精度、低多边形 |
| `Core/Localization/`、`Runtime/UILocalizationBridge.cs` | 92 多语言 |
| `Core/Theming/`、`UILocalizationBridge` | 93 暗/亮主题 |
| `Core/Tutorial/` | 94 内置教程 |

## 快速开始

1. 新建 Unity 场景，创建一个空 GameObject。
2. 挂上 `CncSceneBootstrap`，按需配置机床类型、毛坯尺寸、材料、初始视角。
3. 运行，然后从你的 UI 代码调用：

```csharp
// 加载 G 代码
bootstrap.LoadProgram(gcodeText);

// 仿真控制
bootstrap.Simulator.Play();
bootstrap.Simulator.SetSpeed(10f);
bootstrap.Simulator.Pause();
bootstrap.Simulator.StepForward();
bootstrap.Simulator.StartFromLine(42);
bootstrap.Simulator.SeekProgress(0.5);

// 视角
bootstrap.Rig.SetView(ViewPreset.Top);
bootstrap.Rig.FrameAll(...); // 或 bootstrap.Simulator.FrameWorkpiece()

// 轨迹样式
bootstrap.Toolpath.ToggleStyle();     // 线框 / 实体
bootstrap.Toolpath.SetVisible(false);

// 截面 / 透明度
bootstrap.CaptureRenderTargets();      // 采集场景渲染器
bootstrap.Section.SetEnabled(true);
bootstrap.Section.SetAxis(SectionView.SectionAxis.Z);
bootstrap.Section.SetOffset(0f);
bootstrap.Transparency.SetOpacity(0.4f);

// 测量
bootstrap.Measurement.SetType(MeasurementTool.MeasureType.Distance);
bootstrap.Measurement.TryPick(screenPos); // 取点
float mm = bootstrap.Measurement.DistanceMm;

// 语言 / 主题
bootstrap.Localization.SetLanguage("en-US");
bootstrap.Localization.ToggleTheme();
```

## 事件（供 UI 订阅）

`CncSimBehaviour`：

- `ProgressChanged(double)` — 仿真进度 0-1，驱动进度条。
- `StateChanged(SimulationState)` — 播放/暂停/停止状态。
- `FrameUpdated(SimulationFrame)` — 每帧刀尖位置、进给、转速、当前段、当前行。
- `Finished()` — 程序执行完成。

`MeasurementTool.Measured(MeasureType, float)` — 测量结果（mm / 度）。

`UILocalizationBridge.LanguageChanged(string)`、`ThemeChanged(ThemeManager)` — 语言/主题刷新。

## 坐标系约定（重要）

- Core 使用右手系、Z 向上、单位 mm。
- Unity 使用左手系、Y 向上。
- `MeshBuilder.ToUnity` 做 `(x, y, z) -> (x, z, y)` 映射，并翻转三角形绕序；`ToCnc` 为逆变换。
- 一切跨层坐标转换都必须经过 `MeshBuilder`，不要在业务代码里手写轴交换。

## 材料去除实时更新

`CncSimBehaviour.LiveRemoval` 打开时，仿真推进过程中按 `RemovalRefreshInterval` 间隔调用
`SceneEntityRenderer.UpdateRemovalMesh()` 刷新去除网格（功能 54）。若追求流畅，可增大间隔或
关闭实时、改为暂停时刷新。

## 性能档位

`CncSimBehaviour.Sim.Performance.ApplyPreset(DetailLevel.LowPoly | Balanced | HighFidelity)`
控制去除分辨率、轨迹弦高、管道侧面数、毛坯分段与网格简化阈值。

## 自定义 Shader 提示

截面视图要求 Shader 读取 `_ClipPlane`（float4：xyz 法线、w 距离）与
`_ClipEnabled`（0/1）。默认会尝试 Standard，如需真正的剖切效果，请在你的 Shader 中做
`clip(dot(worldPos, _ClipPlane.xyz) - _ClipPlane.w)`。
