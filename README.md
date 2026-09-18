# AdventureGame_2

> **Unreal Engine 5.8** · 第三人称障碍闯关（跑酷） · 纯蓝图实现 · 🚧 开发中

一个用于学习 UE5 游戏开发流程的练习项目，目标是打通「角色控制 → 动画系统 → 关卡交互 → 死亡重生 → 玩法闭环」的完整链路，并熟悉 UE5 的 Gameplay 框架、Enhanced Input 输入体系与蓝图资源组织方式。

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

本项目实现了一套完整的第三人称闯关玩法框架：使用 Enhanced Input 处理移动、视角与滚轮缩放，用动画蓝图状态机驱动 Idle / Run / 跳跃三态切换，在关卡中布置了会移动、旋转、弹射与致命的多种机关，并实现了「死亡 → Ragdoll → 检查点重生」的失败循环。

## 玩法与操作

**玩法概述**

玩家以第三人称视角操控角色在户外场景中奔跑、跳跃，借助地形与场景机关（移动平台、旋转摆锤、发射器等）越过障碍、推进闯关路线。

- **移动平台**：在预设的两个位置之间往返移动，需要踩准时机通过
- **旋转摆锤**：持续旋转的球体障碍，触碰即死
- **旋转机关**：持续自转的障碍，触碰会被击飞并死亡
- **发射器（跳板）**：踩上去会把角色朝指定方向弹射出去
- **检查点**：触碰后记录重生位置，并以材质变色作为反馈

死亡流程：角色掉落关卡下方或触碰致命机关 → 进入死亡状态（Ragdoll 物理表现）→ 延时销毁角色 → 由 GameMode 在**最近的检查点**重新生成角色。

> ⚠️ 完整的胜利条件与结算流程仍在开发中，详见 [开发进度](#开发进度)。

**操作方式**

| 操作 | 输入 | 对应资产 |
|------|------|----------|
| 前后左右移动 | `W` `A` `S` `D` | `IA_Move`（Axis2D） |
| 旋转视角 | 鼠标移动 | `IA_Look`（Axis2D） |
| 跳跃 | `Space` | `IA_Jump`（Boolean） |
| **缩放视角远近** | **鼠标滚轮** | `IA_CameraZoom`（Axis1D） |

全部按键映射集中定义在 `IMC_Adventure`（Input Mapping Context）中，便于统一修改与后续扩展手柄支持。

## 技术点

### 1. Enhanced Input 输入体系

- 采用 UE5 的 Enhanced Input 插件（`DefaultPlayerInputClass` / `DefaultInputComponentClass` 已在项目配置中切换为 Enhanced 版本），替代旧版 Axis/Action 映射。
- 定义四个 Input Action：`IA_Move`、`IA_Look` 为 **Axis2D**，`IA_Jump` 为 **Boolean**，`IA_CameraZoom` 为 **Axis1D**。
- `IMC_Adventure` 中通过 **Input Modifier** 完成轴向修正：
  - `InputModifierSwizzleAxis` —— 交换轴向，把单轴按键转为 2D 向量；
  - `InputModifierNegate` —— 取反，实现 S 键后退与 A 键左移。
- 滚轮缩放绑定到 `Mouse Wheel Axis`（引擎注册的 Axis1D 轴键）。**注意不要用 `Mouse Wheel Up` / `Mouse Wheel Down`** —— 它们是布尔键，配 Axis1D 动作会恒定输出 `1.0`，无法区分滚动方向。
- 角色 `BeginPlay` 时通过 `GetLocalPlayerSubSystemFromPlayerController` 获取 `EnhancedInputLocalPlayerSubsystem`，再调用 `AddMappingContext` 注册映射上下文。

### 2. 第三人称角色架构

- 角色继承自引擎 `Character` 类，组件结构为：
  `CapsuleComponent`（碰撞） → `SkeletalMeshComponent`（角色网格） → `SpringArmComponent`（弹簧臂，开启 `bDoCollisionTest` 防止穿墙） → `CameraComponent`（跟随相机）。
- 移动逻辑：`IA_Move` 的 Axis2D 值经 `BreakVector2D` 拆分为 X / Y 分量，分别与 `GetForwardVector` / `GetRightVector` 相乘后交给 `AddMovementInput`，实现相机朝向相关的移动方向。
- 视角逻辑：`IA_Look` 的 Axis2D 值分别交给 `AddControllerYawInput` / `AddControllerPitchInput`。
- **滚轮缩放**：`IA_CameraZoom` 的输入值（Axis1D，引脚类型本身即为 double，无需转换节点）参与如下运算后写回弹簧臂：

  ```
  缩放步长 = 输入值 × ZoomStep
  新臂长   = FClamp(当前臂长 − 缩放步长, MinArmLength, MaxArmLength)
  → SpringArm.SetTargetArmLength
  ```

  用「减法」而非加法，使滚轮上滚为拉近、下滚为拉远，无需额外取反节点。
- 死亡处理：切换到 `MOVE_None` 移动模式并开启 `SetSimulatePhysics`，让角色转为 Ragdoll 物理表现。

### 3. 动画蓝图状态机

- `ABP_AdventureCharacter` 继承自 `AnimInstance`，通过事件图表计算并暴露状态机所需参数：
  - `Speed` —— 角色速度向量的 `VSizeXY`（水平速度大小），驱动 1D 混合空间；
  - `Z Speed` —— 速度的 Z 分量，配合 `IsFalling` 判断跳跃阶段。
- 状态机包含 `locomotion`、`Jump_start`、`Jump_loop`、`Jump_end` 状态，由比较与布尔逻辑（`Greater` / `LessEqual` / `BooleanAND` / `BooleanOR`）驱动状态转移。
- 过渡条件使用了 `GetRelevantAnimTimeRemaining`（获取当前动画剩余时间），实现「起跳动画播完 → 空中循环 → 落地」的时序衔接。
- 图内混合使用 `AnimGraphNode_BlendSpacePlayer`（移动）与 `AnimGraphNode_SequencePlayer`（跳跃动作）。

### 4. 1D 混合空间（Blend Space）

- `AS1D_Adventure` 为 **BlendSpace1D**，混合参数（`BlendParameter`）为 `Speed`。
- 采样点：`MM_Idle`（速度 0） ↔ `MM_Run_Fwd`（最高速度），插值类型为 `BSIT_Cubic`，实现待机到奔跑的平滑过渡。

### 5. 失败判定与检查点重生

一套完整的「难度循环」，由三个部分协作完成：

- **失败判定**：`BP_AdventureCharacter` 的 `Event Tick` 中取角色世界坐标 Z，与阈值比较；低于阈值即调用自定义事件 `Dead`（掉落死亡）。此外，所有致命机关的 Overlap 事件也会直接调用 `Dead`。
- **死亡表现**：`Dead` 事件依次执行 —— 网格开启物理模拟（Ragdoll）→ 弹簧臂关闭碰撞检测（`bDoCollisionTest = False`，避免尸体抖动）→ 移动模式设为 `MOVE_None` → `Delay` 延时 → `DestroyActor` → 调用 GameMode 的 `Rebirth`。
- **检查点重生**：`BP_AdventureMode` 持有变量 `RebirthLocation`，自定义事件 `Rebirth` 会执行 `SpawnActor(BP_AdventureCharacter)` 并 `Possess` 到新角色上；`CheckPoint` 机关在玩家 Overlap 时把自身位置写入 `RebirthLocation`，并用 `SetColorParameterValueOnMaterials` 改变材质颜色作为视觉反馈。

### 6. 可交互机关

所有机关都遵循同一套模式：`Actor` 派生 → 碰撞体 + 网格 → `OnComponentBeginOverlap` → **动态 Cast 到 `BP_AdventureCharacter`** 确认是玩家后再执行效果。差别只在运动方式与触发结果：

| 蓝图 | 运动方式 | 触发效果 |
|------|----------|----------|
| `BP_Sphere` | Timeline 浮点曲线驱动旋转 | 玩家死亡 |
| `BP_002` | Timeline 在两个预设位置之间插值移动 | ——（移动平台） |
| `BP_003` | `AddRelativeRotation` 每帧自转 | 击飞（`AddImpulse`）并死亡 |
| `BP_004` | Timeline 驱动 `SetRelativeTransform` | 玩家死亡 |
| `BP_Launch` | 静态 | `LaunchCharacter` 弹射角色 |
| `CheckPoint` | 静态 | 记录重生点 + 材质变色 |
| `NewBlueprint` | 静态 | 遍历并激活 Niagara 特效组件 |

### 7. 渲染与项目配置

- **Lumen** 动态全局光照 + 反射（`DynamicGlobalIlluminationMethod=1`）
- **虚拟阴影贴图**（Virtual Shadow Maps）
- **光线追踪** 与 Substrate 材质系统
- 静态光照已关闭（`r.AllowStaticLighting=False`），全部走动态光照流程
- 图形 RHI：**DX12 / Shader Model 6**
- 默认地图已指向项目关卡，启动游戏即进入 `Level_Scene_01`

## 蓝图结构说明

```
Content/Code/                          ← 自建蓝图（本项目核心逻辑）
│
├── Character/
│   ├── BP_AdventureCharacter         # 角色蓝图（父类: Engine.Character）
│   │                                   #  组件: Capsule / Mesh / SpringArm / Camera
│   │                                   #  职责: 输入绑定、移动与视角、滚轮缩放、
│   │                                   #        掉落死亡判定、死亡 Ragdoll 处理
│   ├── BP_AdventureMode              # GameMode（父类: Engine.GameModeBase）
│   │                                   #  变量: RebirthLocation（重生点）
│   │                                   #  职责: 指定 DefaultPawnClass，
│   │                                   #        提供 Rebirth 事件（生成角色 + Possess）
│   ├── Admition/                      # 动画相关
│   │   ├── ABP_AdventureCharacter     # 动画蓝图（父类: Engine.AnimInstance）
│   │   │                               #  变量: Speed / Z Speed / IsFalling
│   │   │                               #  职责: 计算并驱动动画状态机
│   │   └── AS1D_Adventure             # 1D 混合空间（Idle ↔ Run，参数: Speed）
│   └── Input/
│       ├── IMC_Adventure              # 输入映射上下文
│       ├── IA_Move                    # Axis2D（W/A/S/D）
│       ├── IA_Look                    # Axis2D（鼠标）
│       ├── IA_Jump                    # Boolean（空格）
│       └── IA_CameraZoom              # Axis1D（鼠标滚轮）
│
└── Mechanism/                         # 机关（均继承自 Engine.Actor）
    ├── BP_Sphere                      # 旋转球体障碍 → 致死
    ├── BP_002                         # 往返移动平台
    ├── BP_003                         # 自转障碍 → 击飞致死
    ├── BP_004                         # Transform 动画障碍 → 致死
    ├── BP_Launch                      # 发射器（弹射角色）
    ├── CheckPoint                     # 检查点（记录重生点 + 变色反馈）
    └── NewBlueprint                   # Niagara 特效触发区
```

**依赖关系**

```
BP_AdventureMode ──DefaultPawnClass──▶ BP_AdventureCharacter
        ▲                                      │
        │                                      ├── AnimClass ──▶ ABP_AdventureCharacter
        │                                      │                      │
        │                                      ├── Enhanced Input      └── 采样 ──▶ AS1D_Adventure
        │                                      │   (IMC_Adventure)
        │                                      │
        │  Rebirth（重新生成 + Possess）◀────── Dead（死亡后调用）
        │
   CheckPoint ──写入 RebirthLocation──┘

致命机关（BP_Sphere / BP_003 / BP_004）──Overlap + Cast──▶ BP_AdventureCharacter.Dead
BP_Launch   ──Overlap + Cast──▶ LaunchCharacter
NewBlueprint──Overlap + Cast──▶ 激活 Niagara 特效
```

配置层面的入口绑定（`Config/DefaultEngine.ini`）：

```ini
[/Script/EngineSettings.GameMapsSettings]
GameDefaultMap=/Game/ChallengeGame/Maps/Level_Scene_01.Level_Scene_01
GlobalDefaultGameMode=/Game/Code/Character/BP_AdventureMode.BP_AdventureMode_C
```

> 所有蓝图的事件图表均已添加**中文范围注释框**，标注每个事件的作用与执行链路，便于复习与后续维护。

## 项目结构

```
AdventureGame_2/
├── AdventureGame_2.uproject      # 项目描述文件（EngineAssociation: 5.8）
├── Config/                       # 项目配置
│   ├── DefaultEngine.ini         #   默认关卡 / GameMode、渲染设置
│   ├── DefaultGame.ini           #   项目信息
│   ├── DefaultInput.ini          #   Enhanced Input 类替换、灵敏度
│   └── DefaultEditor.ini
├── Content/
│   ├── Code/                     # 自建蓝图（见上一节）
│   ├── ChallengeGame/            # 关卡与场景美术
│   │   ├── Maps/                 #   Level_Scene_01.umap（主关卡）
│   │   ├── Meshes/               #   56 个静态网格（跳板/齿轮/木桥/气球等）
│   │   ├── Materials/            #   66 个材质与材质实例
│   │   └── Textures/             #   55 张贴图（地表/植物/水面/噪声等）
│   └── FestivalFX/               # 节日特效素材包（Niagara 特效/材质/网格）
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

4. **运行**

   项目默认关卡已配置为 `Level_Scene_01`，直接按工具栏的 **Play**（或 `Alt + P`）即可开始游玩。

   如需在编辑器中查看其它关卡（如素材包自带的 Demo 场景），从内容浏览器手动打开即可。

## 开发进度

项目当前处于**开发中**状态。

**已完成**

- [x] 项目初始化与渲染管线配置（Lumen / VSM / DX12-SM6）
- [x] 默认关卡与 GameMode 入口配置
- [x] Enhanced Input 体系搭建（4 个 Input Action + 1 个 Mapping Context，含轴向修正）
- [x] 第三人称角色：弹簧臂相机、移动、视角控制
- [x] **鼠标滚轮缩放视角**（Axis1D 输入 + FClamp 范围限制）
- [x] 动画蓝图状态机（Idle / Run / Jump 三态）+ 1D 混合空间
- [x] 角色死亡状态与 Ragdoll 表现
- [x] **失败判定**：掉落死亡（Tick 中 Z 坐标阈值判断）
- [x] **检查点与重生系统**：CheckPoint 记录重生点 → GameMode.Rebirth 重新生成角色
- [x] 机关系统（7 个）：移动平台、旋转摆锤、自转障碍、Transform 动画障碍、发射器、检查点、特效触发区
- [x] 关卡场景搭建（地形、植被、水面、机关布置）
- [x] Niagara 特效素材导入（FestivalFX）
- [x] 全部蓝图事件图表补齐中文范围注释

**进行中 / 待完成**

- [ ] 关卡的胜利条件与结算流程
- [ ] 完整的 UI（主菜单、HUD 显示检查点状态、结算界面）
- [ ] 音效与视觉反馈打磨
- [ ] 关卡流程与数值调优（跳跃高度、机关难度曲线）
- [ ] **后续计划：将核心逻辑用 C++ 重构**，作为 UE C++ 开发的学习实践

## 已知限制

- **纯蓝图项目**：当前没有任何 C++ 模块（无 `Source/` 目录），逻辑全部由蓝图实现。
- **素材依赖**：如前所述，克隆仓库后必须自行补齐 RPG_Character 素材包，否则角色与动画缺失。
- **未打包**：项目尚未打包为可执行文件，需要 UE 编辑器运行。
- **仓库体积**：目前约 350 MB（含 FestivalFX 特效素材），接近需要启用 Git LFS 的规模。

## 第三方素材

| 素材 | 来源 | 说明 |
|------|------|------|
| RPG_Character | Epic Games 商城素材包 | 角色模型 `SK_Adventurer` 与全部动画序列；**未纳入仓库** |
| FestivalFX | Epic Games 商城素材包 | Niagara 特效、材质、网格；已纳入仓库 |
| ChallengeGame 场景资源 | 课程/教程配套资源 | 关卡、静态网格、材质与贴图 |

第三方素材版权归原作者所有，本项目仅用于学习与个人练习，不作商业用途。

---

**项目性质说明**：这是一个学习阶段的练习项目，用于实践 UE5 蓝图开发的完整流程。代码与实现会持续迭代，欢迎交流指正。
