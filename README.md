# AdventureGame_2

> **Unreal Engine 5.8** · 第三人称障碍闯关（跑酷） · 纯蓝图实现 · 🚧 开发中

一个用于学习 UE5 游戏开发流程的练习项目，目标是打通「角色控制 → 动画系统 → 关卡交互 → 玩法闭环」的完整链路，并熟悉 UE5 的 Gameplay 框架、Enhanced Input 输入体系与蓝图资源组织方式。

---

## 目录

- [项目简介](#项目简介)
- [玩法与操作](#玩法与操作)
- [技术点](#技术点)
- [蓝图结构说明](#蓝图结构说明)
- [项目结构](#项目结构)
- [如何运行](#如何运行)
- [开发进度](#开发进度)
- [已知限制](#已知限制)
- [第三方素材](#第三方素材)

---

## 项目简介

| 项目 | 说明 |
|------|------|
| 引擎版本 | Unreal Engine **5.8** |
| 实现方式 | **纯蓝图（Blueprint）**，当前无 C++ 代码 |
| 游戏类型 | 第三人称障碍闯关 / 跑酷 |
| 目标平台 | Windows（DX12 / Shader Model 6） |
| 项目定位 | 学习阶段练习项目，跟随课程/教程实现并自行扩展 |

本项目实现了一个完整的第三人称角色控制方案：使用 Enhanced Input 处理移动与视角输入，用动画蓝图状态机驱动 Idle / Run / 跳跃三态切换，并在关卡中放置可交互的旋转球体机关作为玩法元素。

## 玩法与操作

**玩法概述**

玩家以第三人称视角操控角色在户外场景中奔跑、跳跃，借助地形与场景机关（跳板、木桥、齿轮、尖刺等）越过障碍、推进闯关路线；触碰场景中的**旋转球体机关**会改变它的旋转方向。角色的死亡状态采用 Ragdoll 物理表现。

> ⚠️ 完整的胜利条件与结算流程仍在开发中，详见 [开发进度](#开发进度)。

**操作方式**

| 操作 | 输入 | 对应资产 |
|------|------|----------|
| 前后左右移动 | `W` `A` `S` `D` | `IA_Move`（Axis2D） |
| 旋转视角 | 鼠标移动 | `IA_Look`（Axis2D） |
| 跳跃 | `Space` | `IA_Jump`（Boolean） |

全部按键映射集中定义在 `IMC_Adventure`（Input Mapping Context）中，便于统一修改与后续扩展手柄支持。

## 技术点

### 1. Enhanced Input 输入体系

- 采用 UE5 的 Enhanced Input 插件（`DefaultPlayerInputClass` / `DefaultInputComponentClass` 已在项目配置中切换为 Enhanced 版本），替代旧版 Axis/Action 映射。
- 定义三个 Input Action：`IA_Move`、`IA_Look` 为 **Axis2D** 类型，`IA_Jump` 为 **Boolean** 类型。
- `IMC_Adventure` 中通过 **Input Modifier** 完成轴向修正：
  - `InputModifierSwizzleAxis` —— 交换轴向，把单轴按键转为 2D 向量；
  - `InputModifierNegate` —— 取反，实现 S 键后退与 A 键左移。
- 角色 `BeginPlay` 时通过 `GetLocalPlayerSubSystemFromPlayerController` 获取 `EnhancedInputLocalPlayerSubsystem`，再调用 `AddMappingContext` 注册映射上下文。

### 2. 第三人称角色架构

- 角色继承自引擎 `Character` 类，组件结构为：
  `CapsuleComponent`（碰撞） → `SkeletalMeshComponent`（角色网格） → `SpringArmComponent`（弹簧臂，开启 `bDoCollisionTest` 防止穿墙） → `CameraComponent`（跟随相机）。
- 移动逻辑：`IA_Move` 的 Axis2D 值经 `BreakVector2D` 拆分为 X / Y 分量，分别与 `GetForwardVector` / `GetRightVector` 相乘后交给 `AddMovementInput`，实现相机朝向相关的移动方向。
- 视角逻辑：`IA_Look` 的 Axis2D 值分别交给 `AddControllerYawInput` / `AddControllerPitchInput`。
- 死亡处理：切换到 `MOVE_None` 移动模式并开启 `SetSimulatePhysics`，让角色转为 Ragdoll 物理表现。

### 3. 动画蓝图状态机

- `ABP_AdventureCharacter` 继承自 `AnimInstance`，通过事件图表计算并暴露状态机所需参数：
  - `Speed` —— 角色速度向量的 `VSizeXY`（水平速度大小），驱动 1D 混合空间；
  - `Z Speed` —— 速度的 Z 分量，配合 `IsFalling` 判断跳跃阶段。
- 状态机包含 `locomotion`、`Jump_start`、`Jump_loop`、`Jump_end` 状态，由 `K2Node_PromotableOperator` 组成的比较与布尔逻辑（`Greater` / `LessEqual` / `BooleanAND` / `BooleanOR`）驱动状态转移。
- 过渡条件使用了 `GetRelevantAnimTimeRemaining`（获取当前动画剩余时间），实现「起跳动画播完 → 空中循环 → 落地」的时序衔接。
- 图内混合使用 `AnimGraphNode_BlendSpacePlayer`（移动）与 `AnimGraphNode_SequencePlayer`（跳跃动作）。

### 4. 1D 混合空间（Blend Space）

- `AS1D_Adventure` 为 **BlendSpace1D**，混合参数（`BlendParameter`）为 `Speed`。
- 采样点：`MM_Idle`（速度 0） ↔ `MM_Run_Fwd`（最高速度），插值类型为 `BSIT_Cubic`，实现待机到奔跑的平滑过渡。

### 5. 可交互机关

- `BP_Sphere` 继承自 `Actor`，组件为 `SphereComponent`（球形碰撞体）+ `StaticMeshComponent`（`SM_Sphere`）。
- 旋转表现由 **Timeline（浮点曲线轨道）** 驱动，输出值经 `MakeRotator` 构造旋转量后调用 `K2_SetRelativeRotation`。
- 交互通过 `OnComponentBeginOverlap` 事件实现，并使用 **动态 Cast** 转换为 `BP_AdventureCharacter` 判断是否为玩家角色，进而通过 `Play` / `Reverse` 切换 Timeline 播放方向，改变球体旋转状态。

### 6. 渲染与项目配置

- **Lumen** 动态全局光照 + 反射（`DynamicGlobalIlluminationMethod=1`）
- **虚拟阴影贴图**（Virtual Shadow Maps）
- **光线追踪** 与 Substrate 材质系统
- 静态光照已关闭（`r.AllowStaticLighting=False`），全部走动态光照流程
- 图形 RHI：**DX12 / Shader Model 6**

## 蓝图结构说明

```
Content/Code/                          ← 自建蓝图（本项目核心逻辑）
│
├── Character/
│   ├── BP_AdventureCharacter         # 角色蓝图（父类: Engine.Character）
│   │                                   #  组件: Capsule / Mesh / SpringArm / Camera
│   │                                   #  职责: 输入绑定、移动与视角、死亡 Ragdoll
│   ├── BP_AdventureMode              # GameMode（父类: Engine.GameModeBase）
│   │                                   #  职责: 指定 DefaultPawnClass = BP_AdventureCharacter
│   ├── Admition/                      # 动画相关
│   │   ├── ABP_AdventureCharacter     # 动画蓝图（父类: Engine.AnimInstance）
│   │   │                               #  职责: 计算 Speed/Z Speed/IsFalling，驱动状态机
│   │   └── AS1D_Adventure             # 1D 混合空间（Idle ↔ Run，参数: Speed）
│   └── Input/
│       ├── IMC_Adventure              # 输入映射上下文（W/A/S/D + 鼠标 + 空格）
│       ├── IA_Move                    # Axis2D
│       ├── IA_Look                    # Axis2D
│       └── IA_Jump                    # Boolean
│
└── Mechanism/
    └── BP_Sphere                      # 球形机关（父类: Engine.Actor）
                                        #  Timeline 驱动旋转 + Overlap 检测玩家切换方向
```

**依赖关系**

```
BP_AdventureMode  ──指定 DefaultPawnClass──▶  BP_AdventureCharacter
                                                      │
                                      AnimClass ──────┼──────▶  ABP_AdventureCharacter
                                                      │                   │
                                      Enhanced Input ─┤           采样 ──▶  AS1D_Adventure
                                      (IMC_Adventure) │
                                                      ▼
BP_Sphere  ──Overlap + Cast 判定玩家──▶  BP_AdventureCharacter
```

配置层面的入口绑定（`Config/DefaultEngine.ini`）：

```ini
[/Script/EngineSettings.GameMapsSettings]
GlobalDefaultGameMode=/Game/Code/Character/BP_AdventureMode.BP_AdventureMode_C
```

## 项目结构

```
AdventureGame_2/
├── AdventureGame_2.uproject      # 项目描述文件（EngineAssociation: 5.8）
├── Config/                       # 项目配置
│   ├── DefaultEngine.ini         #   默认 GameMode、渲染设置
│   ├── DefaultGame.ini           #   项目信息
│   ├── DefaultInput.ini          #   Enhanced Input 类替换、灵敏度
│   └── DefaultEditor.ini
├── Content/
│   ├── Code/                     # 自建蓝图（见上一节）
│   └── ChallengeGame/            # 关卡与场景美术
│       ├── Maps/                 #   Level_Scene_01.umap（主关卡）
│       ├── Meshes/               #   56 个静态网格（跳板/齿轮/木桥/气球等）
│       ├── Materials/            #   66 个材质与材质实例
│       └── Textures/             #   55 张贴图（地表/植物/水面/噪声等）
└── .gitignore                    # 排除编译产物与第三方素材包
```

## 如何运行

### 环境要求

- **Unreal Engine 5.8**
- Windows 10/11
- 支持 **DX12 / Shader Model 6** 的显卡（项目启用了 Lumen、虚拟阴影贴图与光线追踪）
- 建议 16 GB 以上内存（首次打开需编译着色器）

### 步骤

1. **克隆仓库**

   ```bash
   git clone https://github.com/Feather-Luo/Adventure_Game.git
   ```

2. **补齐第三方素材包（必需）**

   角色模型与动画资源来自 **RPG_Character** 素材包，该素材包体积为 1.2 GB，**未纳入本仓库**。克隆后需先从 Epic Games Launcher 的「库」中下载该素材包，并添加到本项目，否则角色网格与动画将缺失。

   涉及的资源路径：

   ```
   /Game/RPG_Character/Assets/Meshes/Adventurer/SK_Adventurer     # 角色网格
   /Game/RPG_Character/Demo/Animations/Manny/MM_Idle              # 待机
   /Game/RPG_Character/Demo/Animations/Manny/MM_Run_Fwd           # 奔跑
   /Game/RPG_Character/Demo/Animations/Manny/MM_Jump              # 起跳
   /Game/RPG_Character/Demo/Animations/Manny/MM_Fall_Loop         # 下落循环
   /Game/RPG_Character/Demo/Animations/Manny/MM_Land              # 落地
   ```

3. **打开项目**

   双击 `AdventureGame_2.uproject`。若提示引擎版本不匹配，选择 5.8 或使用「Select Another Version」手动指定。

4. **打开关卡并运行**

   在内容浏览器中打开 `Content/ChallengeGame/Maps/Level_Scene_01`，然后点击工具栏的 **Play**（或按 `Alt + P`）。

## 开发进度

项目当前处于**开发中**状态。

**已完成**

- [x] 项目初始化与渲染管线配置（Lumen / VSM / DX12-SM6）
- [x] Enhanced Input 体系搭建（3 个 Input Action + 1 个 Mapping Context，含轴向修正）
- [x] 第三人称角色：弹簧臂相机、移动、视角控制
- [x] 动画蓝图状态机（Idle / Run / Jump 三态）+ 1D 混合空间
- [x] 角色死亡状态与 Ragdoll 表现
- [x] 球形交互机关（Timeline 旋转 + Overlap 检测 + 方向切换）
- [x] 关卡场景搭建（地形、植被、水面、机关道具）

**进行中 / 待完成**

- [ ] 关卡的胜利条件与结算流程
- [ ] 失败判定（掉落 / 尖刺等危险区域）
- [ ] 完整的 UI（主菜单、HUD、结算界面）
- [ ] 音效与视觉反馈打磨
- [ ] 关卡流程与数值调优
- [ ] **后续计划：将核心逻辑用 C++ 重构**，作为 UE C++ 开发的学习实践

## 已知限制

- **纯蓝图项目**：当前没有任何 C++ 模块（无 `Source/` 目录），逻辑全部由蓝图实现。
- **默认地图配置**：`Config/DefaultEngine.ini` 中的 `GameDefaultMap` 目前指向引擎自带模板地图（`/Engine/Maps/Templates/OpenWorld`），需要通过编辑器启动或手动打开 `Level_Scene_01`；后续会修正为项目自身的关卡。
- **素材依赖**：如前所述，克隆仓库后必须自行补齐 RPG_Character 素材包。
- **未打包**：项目尚未打包为可执行文件，需要 UE 编辑器运行。

## 第三方素材

| 素材 | 来源 | 说明 |
|------|------|------|
| RPG_Character | Epic Games 商城素材包 | 角色模型 `SK_Adventurer` 与全部动画序列；**未纳入仓库** |
| ChallengeGame 场景资源 | 课程/教程配套资源 | 关卡、静态网格、材质与贴图 |

第三方素材版权归原作者所有，本项目仅用于学习与个人练习，不作商业用途。

---

**项目性质说明**：这是一个学习阶段的练习项目，用于实践 UE5 蓝图开发的完整流程。代码与实现会持续迭代，欢迎交流指正。
