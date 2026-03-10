# 士兵双腿平衡机制与 Godot 实现指南

## 概述

本文档回答两个问题：

1. **士兵两腿是如何站立/维持平衡的？** ——从 Cortex Command 的实现机制出发，解析双腿站立平衡的核心原理。
2. **如果用 Godot 实现这种士兵，应该怎么做？** ——提供完整的 GDScript 实现思路与代码示例。

---

## 第一部分：双腿站立与平衡机制解析

Cortex Command 中士兵的双腿平衡并**不是**靠预制关键帧动画实现的，而是由以下四个相互配合的机制共同维持的：

### 1.1 旋转弹簧（Rotational Spring）——最核心的平衡机制

这是双腿站立平衡的根本。在 `AHuman::Update()` 中，每帧对士兵身体的旋转角施加一个"弹簧恢复力"：

```cpp
// Source/Entities/AHuman.cpp ~ 第 2555 行
if (m_Status == STABLE) {
    // 目标角度：直立时接近 0，受瞄准角和蹲伏量轻微调整
    float rotTarget = GetRotAngleTarget(m_MovementState) * ...;

    float rotDiff = rot - rotTarget;       // 当前角度与目标角度的差值
    // 阻尼弹簧：角速度以 0.98 衰减，同时向目标方向施加修正
    m_AngularVel = m_AngularVel * (0.98F - 0.06F * (m_Health / m_MaxHealth))
                 - (rotDiff * 0.5F);
}
```

**物理含义：**
- 若身体向右倾斜（`rotDiff > 0`），弹簧施加负的角加速度，将身体拉回直立
- 角速度每帧衰减 2%（阻尼），防止无限振荡
- 生命值低时阻尼减小，模拟受伤后摇晃更剧烈的状态

**这就是"平衡"的本质**：不是两条腿真正地动态平衡（像真实物理中的倒立摆），而是一个带阻尼的旋转弹簧每帧把身体拽回目标姿态。

### 1.2 脚部位置固定（STAND 路径）——双脚支撑点

站立时（`STAND` 状态），两条腿的 `AtomGroup` 仍然每帧调用 `PushAsLimb()`，但使用的是 STAND 路径——路径仅有一个固定点，脚保持在身体下方预设位置：

```cpp
// Source/Entities/AHuman.cpp ~ 第 2285 行
else if (m_MovementState == STAND) {
    // 前景腿：脚保持在髋关节正下方
    m_pFGFootGroup->PushAsLimb(
        m_Pos + m_pFGLeg->GetParentOffset(),  // 髋关节世界坐标
        m_pFGLeg->GetMaxLength(),              // 腿的最大伸展长度
        m_Vel,
        m_WalkAngle[FGROUND],                  // 地形坡度角
        m_Paths[FGROUND][STAND],               // STAND 路径（固定点）
        deltaTime,
        nullptr,
        !m_pBGLeg                              // 单腿时影响旋转
    );
    // 背景腿类似
}
```

脚的 `AtomGroup` 碰到地面就停下，地面对脚的支持力（法向反作用力）通过 `AddImpulseForce()` 传递给躯体，抵消重力，使士兵保持在地面上。

### 1.3 稳定性阈值（StableVelocityThreshold）——何时进入倒地状态

在 `Actor::Update()` 中检测稳定性：

```cpp
// Source/Entities/Actor.cpp ~ 第 1182 行
if (m_Status == STABLE) {
    // 速度超过阈值 → 不稳定
    if (std::abs(m_Vel.m_X) > m_StableVel.m_X
     || std::abs(m_Vel.m_Y) > m_StableVel.m_Y) {
        m_Status = UNSTABLE;
    }
}
```

- `STABLE` 状态：旋转弹簧拉向直立（上面 1.1 的逻辑）
- `UNSTABLE` 状态：弹簧改为拉向水平倒地
- 倒地后，手脚切换为 `FlailAsLimb()`（被动随身体甩动的布娃娃物理）

### 1.4 腿部程序化动画（Leg IK）——视觉上的"有腿感"

`Leg` 类根据脚踝目标位置，每帧程序化计算腿的旋转角和精灵帧：

```
关节（髋关节）
    │
    │  腿的长度 = |脚踝目标 - 髋关节|
    │  腿的角度 = atan2(脚踝Y - 髋Y, 脚踝X - 髋X)
    ↓
脚踝（AtomGroup 的当前位置）
```

这不是逆运动学（IK）——Cortex Command 的腿部没有膝关节，只是一根可伸缩的"杆"旋转到脚踝方向。

### 1.5 平衡机制总结图

```
重力（向下）
    ↓
躯体（RigidBody）
    │
    ├── 旋转弹簧 ──────────────────── 每帧将身体角度拉回 0（直立）
    │
    ├── 前景脚 AtomGroup ──── 碰地 → 地面支持力 → AddImpulseForce → 躯体上
    │
    └── 背景脚 AtomGroup ──── 碰地 → 地面支持力 → AddImpulseForce → 躯体上
```

两条腿的脚踩在地面上（支持力抵消重力），旋转弹簧维持直立姿态——这就是"站立平衡"的全部。

---

## 第二部分：在 Godot 中实现相同效果

### 2.1 架构设计

在 Godot 中，推荐使用 `CharacterBody2D`（而非 `RigidBody2D`）作为士兵主体，这样可以更方便地控制移动逻辑。腿部使用程序化 IK 或路径点驱动。

```
Soldier (CharacterBody2D)
├── CollisionShape2D          ← 躯体碰撞
├── Sprite2D (body)           ← 躯体精灵
├── FGLeg (Node2D)            ← 前景腿节点
│   ├── Sprite2D (upper_leg)  ← 大腿精灵
│   └── Sprite2D (foot)       ← 脚部精灵
├── BGLeg (Node2D)            ← 背景腿节点
│   ├── Sprite2D (upper_leg)
│   └── Sprite2D (foot)
├── RayCast2D (fg_ground)     ← 前景腿地面检测
├── RayCast2D (bg_ground)     ← 背景腿地面检测
└── RayCast2D (ceiling)       ← 天花板检测（蹲伏用）
```

### 2.2 旋转弹簧（平衡核心）

在 Godot 中用纯脚本模拟旋转弹簧：

```gdscript
# soldier.gd

extends CharacterBody2D

# 旋转弹簧参数
const ROTATION_SPRING_STIFFNESS := 0.5   # 弹簧刚度（对应 CC 中的 0.5）
const ROTATION_SPRING_DAMPING := 0.98    # 角速度阻尼（对应 CC 中的 0.98）

# 稳定性
const STABLE_VEL_X := 15.0              # X 速度阈值（超出则不稳定）
const STABLE_VEL_Y := 25.0              # Y 速度阈值

enum Status { STABLE, UNSTABLE, DEAD }
var status := Status.STABLE
var angular_velocity := 0.0              # 旋转角速度（rad/s）
var rotation_target := 0.0              # 目标旋转角（直立 = 0）

func _physics_process(delta: float) -> void:
    _update_stability()
    _update_rotation_spring(delta)
    _move_and_slide_custom(delta)
    _update_legs(delta)

func _update_stability() -> void:
    if status == Status.STABLE:
        if abs(velocity.x) > STABLE_VEL_X or abs(velocity.y) > STABLE_VEL_Y:
            status = Status.UNSTABLE
    elif status == Status.UNSTABLE:
        if abs(velocity.x) <= STABLE_VEL_X and abs(velocity.y) <= STABLE_VEL_Y:
            status = Status.STABLE

func _update_rotation_spring(delta: float) -> void:
    var rot := rotation

    # 将旋转限定在 [-π, π] 范围内，并检测是否超过半圈（倒置）
    while rot > PI:
        rot -= TAU
    while rot < -PI:
        rot += TAU
    # 超过半圈（即翻转）→ 进入不稳定状态
    if abs(rot) > PI * 0.75:
        status = Status.UNSTABLE

    if status == Status.STABLE:
        var rot_diff := rot - rotation_target
        # 阻尼弹簧：衰减角速度并施加弹簧恢复力
        angular_velocity = angular_velocity * ROTATION_SPRING_DAMPING \
                         - rot_diff * ROTATION_SPRING_STIFFNESS
    elif status == Status.UNSTABLE:
        # 不稳定时倒向前方
        var fall_target := -PI / 2.0 if velocity.x > 1.0 else (PI / 2.0 if rot > 0 else -PI / 2.0)
        var rot_diff := fall_target - rot
        if abs(rot_diff) > 0.1 and abs(rot_diff) < PI:
            angular_velocity += rot_diff * 3.0 * delta

    rotation += angular_velocity * delta
```

### 2.3 腿部 IK（程序化腿部动画）

#### 方案 A：简单的单段腿（无膝关节，类似 CC）

每条腿只是一根从髋关节到脚踝的"杆"，旋转和缩放均程序化计算：

```gdscript
# leg.gd（挂在 FGLeg / BGLeg 节点上）

extends Node2D

@export var max_leg_length := 32.0       # 腿的最大像素长度
@export var hip_offset := Vector2(0, 8)  # 髋关节相对于躯体中心的偏移

@onready var upper_sprite: Sprite2D = $UpperLeg
@onready var foot_sprite: Sprite2D = $Foot

var foot_target := Vector2.ZERO          # 脚踝的世界坐标目标（由 FootController 设置）

func update_visuals(hip_world_pos: Vector2, ankle_target: Vector2) -> void:
    foot_target = ankle_target
    var ankle_vec := foot_target - hip_world_pos  # 髋→脚踝向量
    var dist := ankle_vec.length()
    dist = clamp(dist, 0.0, max_leg_length)

    # 腿部旋转角 = 髋→脚踝方向
    upper_sprite.rotation = ankle_vec.angle()
    # 腿部长度缩放（如果精灵原始长度 = max_leg_length）
    upper_sprite.scale.x = dist / max_leg_length

    # 脚部跟随脚踝位置
    foot_sprite.global_position = foot_target
    # 站立/支撑时脚保持水平；抬脚时脚垂直于腿
    if ankle_vec.y > 0:          # 脚主要向下（支撑阶段）
        foot_sprite.rotation = 0.0
    else:                         # 脚抬起（摆动阶段）
        foot_sprite.rotation = upper_sprite.rotation + PI / 2.0
```

#### 方案 B：两段腿（含膝关节 IK）

如果想要更真实的膝关节效果，使用余弦定律求解 IK：

```gdscript
# two_segment_leg.gd

extends Node2D

@export var upper_length := 16.0    # 大腿长度（像素）
@export var lower_length := 16.0    # 小腿长度（像素）
@export var knee_direction := 1.0   # 膝盖朝向：1=向前，-1=向后

@onready var upper_sprite: Sprite2D = $UpperLeg
@onready var lower_sprite: Sprite2D = $LowerLeg
@onready var foot_sprite: Sprite2D  = $Foot

func solve_ik(hip_world: Vector2, foot_world: Vector2) -> void:
    var diff := foot_world - hip_world
    var dist := diff.length()
    dist = clamp(dist, abs(upper_length - lower_length) + 0.01,
                       upper_length + lower_length - 0.01)

    # 余弦定律求膝关节角度
    var cos_angle_at_hip := (dist * dist + upper_length * upper_length
                              - lower_length * lower_length) \
                            / (2.0 * dist * upper_length)
    cos_angle_at_hip = clamp(cos_angle_at_hip, -1.0, 1.0)
    var angle_at_hip := acos(cos_angle_at_hip)

    # 基准角（髋→脚方向）
    var base_angle := diff.angle()

    # 膝盖偏向：knee_direction 控制膝盖朝前或朝后
    var hip_angle  := base_angle - angle_at_hip * knee_direction
    var knee_world := hip_world + Vector2(upper_length, 0).rotated(hip_angle)

    var lower_vec  := foot_world - knee_world
    var knee_angle := lower_vec.angle()

    # 更新精灵
    upper_sprite.global_position = hip_world
    upper_sprite.rotation = hip_angle

    lower_sprite.global_position = knee_world
    lower_sprite.rotation = knee_angle

    foot_sprite.global_position = foot_world
    foot_sprite.rotation = 0.0
```

### 2.4 LimbPath 等价实现（路径点脚部控制器）

CC 中的 `LimbPath` 是一组路径点，脚沿着这些点循环移动。在 Godot 中用以下思路实现：

```gdscript
# foot_controller.gd
# 挂在 Soldier 节点上，管理前/背景腿的脚部目标位置

extends Node

# 行走路径点（本地坐标，相对髋关节）
# 模拟 CC 的 LimbPath segments
const WALK_PATH_FG := [
    Vector2(-8,  16),   # 踏地起点
    Vector2( 4,  16),   # 支撑中期
    Vector2(12,  16),   # 蹬地末期
    Vector2( 0, -4),    # 抬脚（摆动相）
    Vector2(-8,  16),   # 回到起点（完成一步）
]
const WALK_PATH_BG := [
    Vector2(-8,  16),
    Vector2( 4,  16),
    Vector2(12,  16),
    Vector2( 0, -4),
    Vector2(-8,  16),
]

# 站立路径点（STAND 路径：只有一个固定点）
const STAND_OFFSET_FG := Vector2(-4, 16)
const STAND_OFFSET_BG := Vector2( 4, 16)

# 路径速度（像素/秒）
const TRAVEL_SPEED := 60.0

var fg_path_progress := 0.0   # 前景腿路径进度 [0, path长度]
var bg_path_progress := 0.0   # 背景腿路径进度（偏移半相位）

# 地面检测射线（需要在 Soldier 节点上设置好）
@onready var fg_ray: RayCast2D = get_parent().get_node("FGGroundRay")
@onready var bg_ray: RayCast2D = get_parent().get_node("BGGroundRay")

func get_foot_target(hip_world: Vector2, path: Array,
                      progress: float, walk_angle: float) -> Vector2:
    """
    根据路径进度返回脚的世界坐标目标。
    walk_angle：地形坡度角（弧度），用于旋转整个路径。
    """
    var total_len := 0.0
    var prev := path[0]
    for i in range(1, path.size()):
        total_len += (path[i] - prev).length()
        prev = path[i]

    var t := fmod(progress, total_len)
    var dist := 0.0
    prev = path[0]
    for i in range(1, path.size()):
        var seg_len := (path[i] - prev).length()
        if dist + seg_len >= t:
            var local_t := (t - dist) / seg_len
            var local_pos := prev.lerp(path[i], local_t)
            # 旋转以匹配地形坡度
            var rotated := local_pos.rotated(walk_angle)
            return hip_world + rotated
        dist += seg_len
        prev = path[i]
    return hip_world + path[-1].rotated(walk_angle)

func update(delta: float, is_walking: bool, walk_angle: float,
            fg_hip: Vector2, bg_hip: Vector2) -> Dictionary:
    """
    每帧调用，返回前/背景腿的脚踝目标坐标。
    """
    var result := {"fg": Vector2.ZERO, "bg": Vector2.ZERO}

    if is_walking:
        fg_path_progress += TRAVEL_SPEED * delta
        bg_path_progress += TRAVEL_SPEED * delta
        result["fg"] = get_foot_target(fg_hip, WALK_PATH_FG, fg_path_progress, walk_angle)
        result["bg"] = get_foot_target(bg_hip, WALK_PATH_BG, bg_path_progress, walk_angle)
    else:
        # 站立：脚保持在固定偏移位置（旋转以匹配地形）
        result["fg"] = fg_hip + STAND_OFFSET_FG.rotated(walk_angle)
        result["bg"] = bg_hip + STAND_OFFSET_BG.rotated(walk_angle)

    # 将脚踝目标吸附到地面（防止脚踏空或嵌入地形）
    result["fg"] = _snap_to_ground(result["fg"], fg_ray)
    result["bg"] = _snap_to_ground(result["bg"], bg_ray)

    return result

func _snap_to_ground(target: Vector2, ray: RayCast2D) -> Vector2:
    """将脚踝目标垂直吸附到地面，类似 CC 的 AtomGroup 碰撞。"""
    if ray.is_colliding():
        var hit := ray.get_collision_point()
        # 只在脚踝目标在地面以下时才吸附
        if target.y > hit.y:
            return Vector2(target.x, hit.y)
    return target
```

### 2.5 地形坡度检测（walk_angle）

CC 中 `UpdateWalkAngle()` 通过从髋关节向下投射射线检测地形坡度。Godot 等价实现：

```gdscript
# 在 soldier.gd 中

@onready var slope_ray_left: RayCast2D  = $SlopeRayLeft   # 左侧地面射线
@onready var slope_ray_right: RayCast2D = $SlopeRayRight  # 右侧地面射线

var walk_angle := 0.0   # 当前地形坡度角（弧度）

func _update_walk_angle() -> void:
    if slope_ray_left.is_colliding() and slope_ray_right.is_colliding():
        var left_hit  := slope_ray_left.get_collision_point()
        var right_hit := slope_ray_right.get_collision_point()
        var slope_vec := right_hit - left_hit
        var target_angle := slope_vec.angle()
        # 平滑插值（避免角度突变）
        walk_angle = lerp_angle(walk_angle, target_angle, 0.2)
    else:
        walk_angle = lerp_angle(walk_angle, 0.0, 0.2)
```

SlopeRayLeft 和 SlopeRayRight 两条射线分别从士兵左右两侧向下投射，用两个碰撞点的连线方向估算地面坡度，再旋转腿部路径使脚自然踏在坡面上。

### 2.6 蹲伏自适应（UpdateCrouching 等价）

```gdscript
# 在 soldier.gd 中

@onready var ceiling_ray: RayCast2D = $CeilingRay
const MAX_CROUCH_SHIFT := 6.0          # 最大蹲伏偏移量（像素）
var crouch_amount := 0.0               # 当前蹲伏量 [0=直立, 1=完全蹲伏]
var walk_path_offset := Vector2.ZERO   # 路径整体偏移

func _update_crouching(delta: float) -> void:
    var desired_offset := 0.0
    if Input.is_action_pressed("crouch"):
        desired_offset = MAX_CROUCH_SHIFT
    elif ceiling_ray.is_colliding():
        var headroom := ceiling_ray.get_collision_point().y - global_position.y
        var needed := -headroom + 16.0  # 16px 为头部空间需求
        desired_offset = clamp(needed, 0.0, MAX_CROUCH_SHIFT)

    # 平滑过渡（对应 CC 的 30% 插值）
    walk_path_offset.y = lerp(walk_path_offset.y, -desired_offset, 0.3)
    if MAX_CROUCH_SHIFT > 0.0:
        crouch_amount = -walk_path_offset.y / MAX_CROUCH_SHIFT
```

### 2.7 完整的 soldier.gd 主循环

```gdscript
# soldier.gd — 完整版

extends CharacterBody2D

const GRAVITY := 800.0
const MOVE_SPEED := 80.0

# 旋转弹簧
const SPRING_STIFFNESS := 0.5
const SPRING_DAMPING := 0.98

enum Status { STABLE, UNSTABLE }
var status := Status.STABLE
var angular_velocity := 0.0

@onready var fg_leg: Node  = $FGLeg
@onready var bg_leg: Node  = $BGLeg
@onready var foot_ctrl     = $FootController

var walk_angle := 0.0

func _physics_process(delta: float) -> void:
    # 1. 重力
    if not is_on_floor():
        velocity.y += GRAVITY * delta

    # 2. 水平移动输入
    var dir := Input.get_axis("move_left", "move_right")
    velocity.x = dir * MOVE_SPEED

    # 3. CharacterBody2D 内置移动（处理碰撞）
    move_and_slide()

    # 4. 更新稳定性
    _update_stability()

    # 5. 旋转弹簧（平衡）
    _update_rotation_spring(delta)

    # 6. 坡度检测
    _update_walk_angle()

    # 7. 蹲伏
    _update_crouching(delta)

    # 8. 计算腿部脚踝目标，更新腿部视觉
    var is_walking := abs(velocity.x) > 1.0
    var fg_hip := global_position + Vector2(-4, 8).rotated(rotation)
    var bg_hip := global_position + Vector2( 4, 8).rotated(rotation)
    var targets := foot_ctrl.update(delta, is_walking, walk_angle, fg_hip, bg_hip)

    fg_leg.update_visuals(fg_hip, targets["fg"])
    bg_leg.update_visuals(bg_hip, targets["bg"])

func _update_stability() -> void:
    if status == Status.STABLE:
        if abs(velocity.x) > 15.0 or abs(velocity.y) > 25.0:
            status = Status.UNSTABLE
    else:
        if abs(velocity.x) <= 15.0 and abs(velocity.y) <= 25.0:
            status = Status.STABLE

func _update_rotation_spring(delta: float) -> void:
    if status == Status.STABLE:
        var rot_diff := rotation - 0.0     # 目标角 = 0（直立）
        angular_velocity = angular_velocity * SPRING_DAMPING \
                         - rot_diff * SPRING_STIFFNESS
    elif status == Status.UNSTABLE:
        var fall_target := -PI / 2.0 if velocity.x > 1.0 else PI / 2.0
        var rot_diff := fall_target - rotation
        if abs(rot_diff) > 0.1 and abs(rot_diff) < PI:
            angular_velocity += rot_diff * 3.0 * delta
    rotation += angular_velocity * delta
```

---

## 第三部分：关键参数速查表

| 参数 | CC 原值 | Godot 等价 | 说明 |
|------|---------|-----------|------|
| 旋转弹簧刚度 | `rotDiff * 0.5` | `SPRING_STIFFNESS = 0.5` | 越大越快回正，过大会抖动 |
| 旋转弹簧阻尼 | `angVel * 0.98` | `SPRING_DAMPING = 0.98` | 越小阻尼越强，越平稳 |
| 稳定速度阈值 X | `m_StableVel.X = 15.0` | `STABLE_VEL_X = 15.0` | 超出则倒地 |
| 稳定速度阈值 Y | `m_StableVel.Y = 25.0` | `STABLE_VEL_Y = 25.0` | 超出则倒地 |
| 最大蹲伏偏移 | `m_MaxWalkPathCrouchShift = 6.0` | `MAX_CROUCH_SHIFT = 6.0` | 蹲伏时路径上移像素数 |
| 路径行进速度 | `LimbPath.TravelSpeed` | `TRAVEL_SPEED = 60.0` | 脚沿路径移动的速度 |
| 坡度平滑率 | 约 20% 插值 | `lerp_angle(..., 0.2)` | walk_angle 平滑响应速度 |

---

## 第四部分：与 CC 实现的差异说明

| 方面 | Cortex Command (C++) | Godot 简化版 |
|------|----------------------|-------------|
| 物理引擎 | 自定义积分 + AtomGroup 碰撞 | Godot 内置 CharacterBody2D |
| 腿部推进力 | AtomGroup 逐像素 Bresenham 碰撞，产生地面反作用力 | RayCast2D 地面检测 + 路径插值 |
| 地形变形 | 脚可以破坏地形 | 标准 TileMap / StaticBody2D 碰撞 |
| 腿部段数 | 单段（可伸缩杆） | 可选单段或双段（含膝关节 IK） |
| 旋转弹簧 | 手动积分角速度 | 手动积分角速度（原理相同） |
| 坡度检测 | 光线投射检测地形材质 | RayCast2D |
| 步伐事件 | 路径重启时触发 `OnStride` Lua 回调 | 可在路径进度归零时 `emit_signal("on_stride")` |

---

## 总结

**士兵双腿站立平衡的核心机制**（三句话版本）：

1. **旋转弹簧**每帧把身体角度拉向目标姿态（直立 = 0），同时对角速度施加阻尼，防止过冲和震荡。
2. **两条腿的脚**通过与地形碰撞产生支持力，传回给躯体，抵消重力，使角色站在地面上而不下沉。
3. 当速度过大时判为**不稳定（UNSTABLE）**，弹簧转为拉向倒地方向，肢体进入布娃娃模式。

**Godot 实现最简路径**：
`CharacterBody2D` + 旋转弹簧脚本 + `RayCast2D` 地面检测 + 路径点脚部控制器 + 单段或双段腿部 IK。
