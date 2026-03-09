# Soldier Movement Animation System

## Overview / 概述

The soldier's two-leg walking movement in Cortex Command is **procedural physics-based animation**, not traditional frame-by-frame sprite animation.

士兵的两脚移动使用的是**程序化物理动画（Procedural Physics-Based Animation）**，而不是传统的逐帧精灵动画。

## How It Works / 工作原理

### LimbPath (肢体路径系统)

Each leg follows a **LimbPath** — a set of waypoint vectors that define a continuous motion path. The foot moves along these waypoints at a configurable travel speed, applying physical forces to the terrain as it pushes off.

每条腿沿着一条 **LimbPath**（肢体路径）运动——由一组路径点向量组成的连续运动路线。脚沿着这些路径点以可配置的速度移动，并在蹬地时对地形施加物理力。

Key properties of each LimbPath:
- **Segments**: An ordered sequence of `Vector` waypoints the foot travels through
- **TravelSpeed**: Movement speed along the path (m/s)
- **PushForce**: Force the foot exerts against the terrain (kg·m/s²)
- **Rotation**: Path rotation adapted to match terrain slope
- **Scale**: Path scale for longer/shorter strides

### Leg Extension (腿部伸展)

The `Leg` class interpolates between a **contracted offset** and an **extended offset** based on where the foot target currently is along the LimbPath. The leg sprite rotates and selects frames based on the current ankle position — this is computed procedurally each frame, not pre-animated.

`Leg` 类根据脚的目标在 LimbPath 上的位置，在**收缩偏移**和**伸展偏移**之间进行插值。腿部精灵的旋转和帧选择基于当前脚踝位置——这是每帧程序化计算的，不是预制动画。

### Terrain Adaptation (地形自适应)

The system adapts to terrain in real-time:

- **Walk Angle**: `UpdateWalkAngle()` rotates the walk path to match the ground slope, so the soldier walks naturally on hills and ramps.
- **Crouch Adaptation**: `UpdateCrouching()` adjusts the walk path height when under low ceilings, allowing the soldier to duck automatically.
- **Foot Collision**: The `AtomGroup` system handles foot-terrain collisions. Foot collisions can be selectively disabled near the end of a stride to prevent the foot from snagging on terrain.

系统实时适应地形：
- **行走角度**：`UpdateWalkAngle()` 根据地面坡度旋转行走路径，使士兵在斜坡上自然行走。
- **蹲伏自适应**：`UpdateCrouching()` 在低矮天花板下自动调整行走路径高度。
- **脚部碰撞**：`AtomGroup` 系统处理脚与地形的碰撞检测。

### Stride Cycle (步伐周期)

When a leg's LimbPath completes its full cycle and restarts, this is detected as a **stride**:
- The `m_StrideFrame` flag is set
- The `OnStride` Lua callback is triggered
- A `StrideSound` plays if configured

The foreground and background legs operate on offset phases to create a natural alternating gait.

当一条腿的 LimbPath 完成整个周期并重新开始时，系统检测为一次**步伐**：
- 设置 `m_StrideFrame` 标志
- 触发 `OnStride` Lua 回调
- 播放步伐音效（如果配置了的话）

前景腿和背景腿以交错相位运行，产生自然的交替步态。

### Arm Swing (手臂摆动)

Arms swing in sync with leg movement:
- `ArmSwingRate` controls the swing magnitude when not holding a device
- `DeviceArmSwayRate` controls weapon sway when holding a device

手臂与腿部运动同步摆动，由 `ArmSwingRate` 和 `DeviceArmSwayRate` 属性控制。

## Architecture / 架构

```
AHuman (humanoid actor)
├── FG Leg (foreground leg) ─── Leg class
├── BG Leg (background leg) ─── Leg class
├── FG Arm (foreground arm) ─── Arm class
├── BG Arm (background arm) ─── Arm class
├── Head ────────────────────── Attachable
├── Body ────────────────────── MOSRotating
└── LimbPath[Layer][MovementState]
    ├── Segments: Vector waypoints
    ├── TravelSpeed: m/s
    ├── PushForce: kg·m/s²
    ├── Rotation: terrain-adaptive
    └── Scale: stride length control
```

## Source Files / 源文件

| File | Description |
|------|-------------|
| `Source/Entities/AHuman.h/cpp` | Humanoid actor — main walking logic, stride detection, terrain adaptation |
| `Source/Entities/Leg.h/cpp` | Leg component — extension/contraction, ankle positioning, frame selection |
| `Source/Entities/LimbPath.h/cpp` | Waypoint path system — segments, speed, force, progress tracking |
| `Source/Entities/AtomGroup.h/cpp` | Physics collision groups for limbs — terrain interaction |
| `Source/Entities/Actor.h/cpp` | Base actor — MovementState enum, common physics |

## Movement States / 移动状态

Each state has its own LimbPath configuration:

| State | Description |
|-------|-------------|
| `STAND` | Standing still (idle leg position) |
| `WALK` | Walking movement |
| `RUN` | Running (faster walk) |
| `CROUCH` | Crouching movement |
| `JUMP` | Jumping |
| `CRAWL` | Crawling while prone |
| `CLIMB` | Climbing terrain |

## Summary / 总结

The soldier movement system is **not traditional animation**. It is a **physics-driven procedural animation system** where:

1. Legs follow waypoint paths (LimbPath) — not pre-drawn frame sequences
2. Leg extension/rotation is calculated each frame based on foot target position
3. The walk path adapts to terrain slope in real-time
4. Feet interact with terrain through physics collisions (AtomGroup)
5. Physical forces are applied to the ground during movement

士兵的移动系统**不是传统动画**，而是**物理驱动的程序化动画系统**：
1. 腿部沿路径点（LimbPath）移动——不是预绘制的帧序列
2. 腿部伸展/旋转每帧根据脚的目标位置计算
3. 行走路径实时适应地形坡度
4. 脚通过物理碰撞（AtomGroup）与地形交互
5. 移动时对地面施加物理力
