# 士兵移动系统技术文档

## 核心问题：士兵是如何移动的？

**结论：士兵的移动既不是直接修改 XY 坐标，也不是简单地施加一个推进力，而是通过「物理驱动的程序化动画系统」实现的。**

具体来说：

1. **腿部沿路径点（LimbPath）运动**，AtomGroup 系统在运动过程中与地形产生碰撞
2. **碰撞产生的反作用力**通过 `AddImpulseForce()` 施加到士兵躯体
3. **物理引擎**将力积分为速度，再将速度积分为位置（`m_Pos += m_Vel * dt`）
4. **腿部的视觉动画**是程序化的：每帧根据脚踝位置计算旋转与精灵帧，而非播放预制动画

---

## 目录

1. [架构总览](#1-架构总览)
2. [移动的本质：力的物理系统](#2-移动的本质力的物理系统)
3. [LimbPath：路径点系统](#3-limbpath路径点系统)
4. [Leg：腿部组件](#4-leg腿部组件)
5. [AtomGroup：碰撞与力的传递](#5-atomgroup碰撞与力的传递)
6. [AHuman：步态控制逻辑](#6-ahuman步态控制逻辑)
7. [完整步伐流程（逐步说明）](#7-完整步伐流程逐步说明)
8. [地形自适应：坡度与蹲伏](#8-地形自适应坡度与蹲伏)
9. [动态速度调整](#9-动态速度调整)
10. [布娃娃物理：不稳定时的肢体](#10-布娃娃物理不稳定时的肢体)
11. [关键数据结构参考](#11-关键数据结构参考)
12. [源文件索引](#12-源文件索引)

---

## 1. 架构总览

```
AHuman（人形角色）
├── m_pFGLeg  ──── Leg（前景腿）
│   └── m_Foot ── Attachable（脚部精灵）
├── m_pBGLeg  ──── Leg（背景腿）
├── m_pFGArm  ──── Arm（前景手臂）
├── m_pBGArm  ──── Arm（背景手臂）
├── m_pHead   ──── Attachable（头部）
├── m_pFGFootGroup ── AtomGroup（前景脚碰撞组）
├── m_pBGFootGroup ── AtomGroup（背景脚碰撞组）
└── m_Paths[2][MOVEMENTSTATECOUNT]
    ├── [FGROUND][STAND]   前景腿·站立路径
    ├── [FGROUND][WALK]    前景腿·行走路径
    ├── [FGROUND][RUN]     前景腿·奔跑路径
    ├── [FGROUND][CRAWL]   前景腿·爬行路径
    ├── [BGROUND][STAND]   背景腿·站立路径
    └── ... (其余状态类推)
```

**各组件职责：**

| 组件 | 职责 |
|------|------|
| `AHuman` | 主控：根据输入选择运动状态，驱动腿部路径，检测步伐，管理旋转弹簧 |
| `LimbPath` | 数据：存储脚的路径点序列，记录行进速度与推力，追踪路径进度 |
| `AtomGroup` | 物理：将脚沿路径推进，与地形逐像素碰撞，将反作用力返回给躯体 |
| `Leg` | 渲染：根据脚踝偏移量计算腿部旋转与精灵帧（程序化动画） |

---

## 2. 移动的本质：力的物理系统

### 2.1 不是直接修改坐标

游戏不会直接写 `m_Pos.m_X += speed`。位置只在每帧末尾由速度积分更新：

```cpp
// MovableObject::Update() 中（伪代码）
m_Vel += accumulatedForces / m_Mass * deltaTime;   // 力 → 速度
m_Pos += m_Vel * deltaTime;                         // 速度 → 位置
```

所有的"推进"都通过改变速度间接实现。

### 2.2 力的来源：脚蹬地面产生反作用力

当士兵行走时：

1. 脚（AtomGroup）被推着沿 LimbPath 路径前行
2. 脚的原子碰到地形时停下，但躯体继续施力
3. 地形对脚的阻力 = 脚对地形的推力的反作用力
4. 这个反作用冲量通过 `AddImpulseForce()` 施加到躯体，驱动躯体前进

```
地面 ──反作用力──▶ 脚 ──冲量──▶ 躯体  ──积分──▶ 位移
      ◀──推力──────                              （每帧）
```

### 2.3 AddImpulseForce 的调用方式

```cpp
// AtomGroup::PushAsLimb() 内部，碰撞计算完成后：
m_OwnerMOSR->AddImpulseForce(
    pushImpulse,                                    // 冲量向量（方向+大小）
    affectRotation ? rotationOffset : Vector()       // 偏移量（非零时产生扭矩）
);
```

- `pushImpulse`：由 `PushTravel()` 计算，取决于地形材料强度和 LimbPath 推力
- `rotationOffset`：当偏移不为零时，冲量会绕该点产生旋转效果（使躯体维持平衡）

---

## 3. LimbPath：路径点系统

`LimbPath` 存储了脚的完整运动路径，由一组有序的向量路径点（segments）组成。

### 3.1 核心属性

```cpp
class LimbPath {
    Vector m_Start;                    // 路径起点（相对关节的本地坐标）
    std::deque<Vector> m_Segments;     // 路径段列表（每段是相对上一点的偏移）
    std::deque<Vector>::iterator m_CurrentSegment;  // 当前所在段

    float m_TravelSpeed;               // 沿路径行进的基础速度（m/s）
    float m_CurrentTravelSpeedMultiplier; // 当前速度乘数（蹲伏/加速时调整）
    Vector m_CurrentScaleMultiplier;   // 当前步幅缩放（快速时步幅变大）

    float m_PushForce;                 // 脚能推动的最大力（kg·m/s²）

    // 关节信息（每帧由 AHuman 设置）
    Vector m_JointPos;                 // 关节世界坐标（髋关节位置）
    Vector m_JointVel;                 // 关节速度
    Matrix m_Rotation;                 // 路径旋转（随地形坡度调整）
    Vector m_PositionOffset;           // 路径整体偏移（蹲伏时向上移动）

    float m_TimeLeft;                  // 本帧剩余处理时间（秒）
    float m_SegProgress;               // 当前段的归一化进度 [0, 1]
    bool m_Ended;                      // 路径是否已完成一个完整周期

    int m_FootCollisionsDisabledSegment; // 从路径末尾起，禁用脚碰撞的段数
                                         // （防止步伐末期脚被地形卡住）
};
```

### 3.2 路径是如何运动的

每帧，`AtomGroup::PushAsLimb()` 会驱动脚沿路径移动：

```
路径点序列（本地坐标，旋转后叠加在关节世界坐标上）：
  Start → Segment[0] → Segment[1] → ... → Segment[N] → （重启）

脚的实际世界目标坐标 = JointPos + Rotation × (Start + ∑Segments[0..k])
```

路径完成（`m_Ended = true`）后，`RestartFree()` 寻找一个不在地形内的起始点重新开始。每次重启表示走完一步。

### 3.3 推力随时间增加

`GetEffectivePushForce()` 会随着路径已经持续时间的增长而增大推力，这样脚被卡住时可以施加更大的力来脱困：

```cpp
float LimbPath::GetEffectivePushForce() const {
    // 路径开始时力较小，随时间线性增长到 m_PushForce 上限
    return std::min(m_PushForce, m_PushForce * (m_PathTimer.GetElapsedSimTimeMS() / 500.0F));
}
```

### 3.4 关键方法

| 方法 | 作用 |
|------|------|
| `GetCurrentVel(limbPos)` | 返回当前路径方向的速度向量（世界坐标） |
| `GetNextTimeChunk(limbPos)` | 返回本帧可用于推进的时间片（秒） |
| `ReportProgress(limbPos)` | 根据脚的当前位置更新路径进度 |
| `GetEffectiveTravelSpeed()` | 有效速度 = TravelSpeed × 乘数 |
| `GetEffectivePushForce()` | 有效推力（随时间增长） |
| `RestartFree(limbPos, ...)` | 在无地形的位置重启路径，返回是否成功 |
| `Terminate()` | 立即将路径标记为完成（强制重启） |
| `GetRegularProgress()` | 路径常规部分进度 [0, 1]，用于前/背景腿同步 |

---

## 4. Leg：腿部组件

`Leg` 类负责腿部精灵的视觉呈现。它不参与物理计算，而是根据 AtomGroup 提供的脚踝位置，**程序化地**计算腿的旋转和精灵帧。

### 4.1 核心属性

```cpp
class Leg {
    Attachable* m_Foot;             // 脚部附件精灵

    // 关节→脚踝 偏移向量（本地坐标）
    Vector m_ContractedOffset;      // 收缩状态（腿最短时关节→脚踝偏移）
    Vector m_ExtendedOffset;        // 伸展状态（腿最长时关节→脚踝偏移）
    Vector m_AnkleOffset;           // 当前踝关节偏移（每帧插值更新）

    float m_MinExtension;           // 腿的最短长度（像素）
    float m_MaxExtension;           // 腿的最长长度（像素）
    float m_NormalizedExtension;    // 当前伸展量 [0=最短, 1=最长]

    Vector m_TargetPosition;        // 脚的目标世界坐标（由 AtomGroup 提供）
    Vector m_IdleOffset;            // 待机时脚相对关节的本地偏移

    float m_MoveSpeed;              // 踝关节追踪目标的速度 [0=不动, 1=立即]
    bool m_WillIdle;                // 目标在关节上方时是否切换为待机偏移
};
```

### 4.2 每帧更新流程

```
Leg::Update()
│
├─ UpdateCurrentAnkleOffset()     ← 将踝关节平滑插值到目标位置
│    │
│    ├─ 计算 targetOffset = ShortestDistance(JointPos, TargetPosition)
│    ├─ 如果目标在关节上方且 m_WillIdle，改用 m_IdleOffset
│    ├─ m_AnkleOffset += (targetOffset - m_AnkleOffset) × m_MoveSpeed
│    └─ ClampMagnitude(m_AnkleOffset, m_MaxExtension, m_MinExtension)
│
├─ UpdateLegRotation()            ← 计算腿部精灵旋转角
│    │
│    ├─ 基础旋转 = m_AnkleOffset 的角度
│    ├─ 计算 m_NormalizedExtension = (|AnkleOffset| - Min) / (Max - Min)
│    └─ 插值补偿旋转（使精灵与收缩→伸展过渡对齐）
│
├─ 更新 m_Frame                   ← 选择腿部精灵帧（代表伸展程度）
│    └─ m_Frame = floor(m_NormalizedExtension × FrameCount)
│
├─ 更新 m_Foot 位置               ← 将脚部附件定位到脚踝处
│    └─ foot.SetParentOffset(AnkleOffset 转换为父坐标)
│
└─ UpdateFootFrameAndRotation()  ← 选择脚部精灵帧与旋转
     │
     ├─ 脚主要向下时（站立/支撑阶段）：
     │    ├─ 根据踝关节水平偏移选帧（0=正下方，1=向前，2=后踏中期，3=向后）
     │    └─ 脚旋转设为 0（水平）
     └─ 腿侧向摆动时（抬脚阶段）：
          └─ 脚旋转 = 腿部旋转 + 90°（脚垂直于腿）
```

### 4.3 踝关节插值代码

```cpp
void Leg::UpdateCurrentAnkleOffset() {
    // 关节到目标的向量（处理场景环绕）
    Vector targetOffset = g_SceneMan.ShortestDistance(m_JointPos, m_TargetPosition, ...);

    // 目标在上方时切换为待机偏移（防止腿向上"反关节"）
    Vector rotatedTarget = targetOffset.GetRadRotatedCopy(m_Parent->GetRotAngle());
    if (m_WillIdle && rotatedTarget.m_Y < -std::abs(rotatedTarget.m_X)) {
        targetOffset = m_Parent->RotateOffset(m_IdleOffset);
    }

    // 平滑插值（线性，每帧靠近 m_MoveSpeed 比例）
    m_AnkleOffset += (targetOffset - m_AnkleOffset) * m_MoveSpeed;

    // 限制腿长在有效范围内
    m_AnkleOffset.ClampMagnitude(m_MaxExtension, m_MinExtension + 0.1F);
}
```

---

## 5. AtomGroup：碰撞与力的传递

`AtomGroup` 是移动系统的物理核心，负责将脚沿路径推进并处理与地形的碰撞。

### 5.1 原子组的概念

一个 AtomGroup 由若干个 **Atom**（原子）组成，每个原子是一个接触点。脚的 AtomGroup 通常只有一到几个原子，代表脚与地面的接触位置。

```cpp
class AtomGroup {
    std::vector<Atom*> m_Atoms;       // 组成该组的原子列表
    MOSRotating* m_OwnerMOSR;         // 所有者（士兵躯体）
    Vector m_LimbPos;                 // 上一帧脚的世界坐标（持久状态）
};
```

### 5.2 PushAsLimb()：主动行走推进

这是行走的核心函数，每帧由 `AHuman::PreControllerUpdate()` 调用。

**函数签名：**

```cpp
bool AtomGroup::PushAsLimb(
    const Vector& jointPos,        // 髋关节世界坐标
    const float limbRadius,        // 腿的最大伸展长度
    const Vector& velocity,        // 士兵当前速度
    const Matrix& rotation,        // 路径旋转矩阵（包含地形坡度）
    LimbPath& limbPath,            // 脚遵循的路径（含路径点、速度、推力）
    const float travelTime,        // 本帧时间（秒）
    bool* restarted,               // [输出] 本帧路径是否完成了一个周期
    bool affectRotation,           // 推力是否影响躯体旋转
    Vector rotationOffset,         // 力的作用点（产生扭矩）
    Vector positionOffset          // 路径整体位移偏移（蹲伏调整）
);
```

**执行流程：**

```
PushAsLimb() 执行流程：

1. 设置 LimbPath 参数
   └─ limbPath.SetJointPos(), SetJointVel(), SetRotation(), SetFrameTime()

2. 检查脚是否偏离路径太远
   └─ 若 Distance(jointPos, m_LimbPos) > ownerRadius × 2，调用 limbPath.Terminate()

3. 主推进循环（循环直到帧时间用完或路径结束）：
   │
   ├─ 若路径已结束：
   │    └─ RestartFree(m_LimbPos, ...) → 找无地形起点重启
   │         └─ *restarted = true（触发步伐事件）
   │
   ├─ 调用 PushTravel()：
   │    ├─ 输入：m_LimbPos（当前位置）、路径速度、最大推力、帧时间片
   │    ├─ 处理：Bresenham 算法逐像素推进脚，检测地形碰撞
   │    └─ 输出：累积冲量、更新后的 m_LimbPos
   │
   └─ limbPath.ReportProgress(m_LimbPos) → 更新路径段进度

4. 处理反向推力（防止被地形推回）
   └─ 若冲量与运动方向相反，将 X 分量衰减为 30%

5. 向躯体施加冲量
   └─ m_OwnerMOSR->AddImpulseForce(pushImpulse, rotationOffset)

6. 将脚的位置告知 Leg 组件
   └─ m_Leg->SetTargetPosition(m_LimbPos)
```

### 5.3 PushTravel()：逐像素碰撞检测

`PushTravel()` 是物理层面的最底层函数，使用 Bresenham 直线算法将脚从当前位置按速度推进，并逐像素检测碰撞。

**核心逻辑（简化）：**

```cpp
Vector AtomGroup::PushTravel(
    Vector& position,        // [输入/输出] 原子组当前世界坐标
    const Vector& velocity,  // 本时间片内的移动速度
    float pushForce,         // 最大推力（N）
    bool& didWrap,
    float travelTime         // 本次推进的时间（秒）
) {
    Vector trajectory = velocity * travelTime * c_PPM;  // 像素轨迹
    // Bresenham 算法参数初始化
    int intPos[2] = { position.FloorX(), position.FloorY() };
    // ...（delta, increment, error 计算）

    Vector returnPush;

    for (int step = 0; step < domainSteps; ++step) {
        // 按主轴步进一个像素
        intPos[dom] += increment[dom];
        if (error >= 0) { intPos[sub] += increment[sub]; error -= delta[dom]; }
        error += delta[sub];

        // 检查每个原子是否碰到地形
        for (Atom* atom : m_Atoms) {
            Vector atomWorldPos = intPos + atom->GetOffset();
            if (g_SceneMan.GetTerrMatter(atomWorldPos) != g_MaterialAir) {
                // 碰到地形：记录碰撞点
                hitAtoms.push_back({ atom, atomWorldPos });
            }
        }

        if (!hitAtoms.empty()) {
            // 计算碰撞法线与推力大小
            Vector normal = CalculateCollisionNormal(hitAtoms);
            float impulse = std::min(pushForce, terrainStrength) * travelTime;
            returnPush += normal * impulse;

            // 将脚推出地形（回退到碰撞前位置）
            position -= normal * penetrationDepth;
            break;  // 停止本次推进
        }
    }

    // 更新 position 到最终到达的位置
    position = nextPos - trajectory.Normalize() * remainingDist;
    return returnPush;
}
```

**关键点：**
- 使用 Bresenham 算法确保不跳过像素，精确检测碰撞
- 每个原子独立检测，支持脚的多点接触
- 碰撞时计算法线方向推力，反向传回给躯体
- 脚被推出地形（不会嵌入地形）

### 5.4 FlailAsLimb()：不稳定时的布娃娃物理

当士兵倒地或死亡时，使用 `FlailAsLimb()` 替代 `PushAsLimb()`：

```cpp
void AtomGroup::FlailAsLimb(
    const Vector& ownerPos,        // 所有者位置
    const Vector& jointOffset,     // 关节偏移（相对所有者）
    const float limbRadius,        // 肢体半径
    const Vector& velocity,        // 所有者速度
    const float angularVel,        // 所有者角速度
    const float limbMass,          // 肢体质量
    const float travelTime         // 帧时间
);
```

区别：
- **不遵循 LimbPath**，肢体完全被动跟随躯体运动
- 根据躯体的线速度和角速度计算肢体的惯性运动
- 碰到地形时仍然产生碰撞力，但不主动推进

---

## 6. AHuman：步态控制逻辑

`AHuman::PreControllerUpdate()` 是每帧驱动整个运动系统的主函数（约 1553–2463 行）。

### 6.1 运动状态枚举

```cpp
// 继承自 Actor::MovementState
enum MovementState {
    NOMOVESTATE = -1,
    STAND = 0,   // 站立（静止）
    WALK,        // 行走
    RUN,         // 奔跑（更快的行走）
    CROUCH,      // 蹲伏
    CRAWL,       // 爬行（俯卧）
    ARMCRAWL,    // 手臂辅助爬行
    CLIMB,       // 攀爬
    JUMP,        // 跳跃
    DISLODGE,    // 自解脱
    MOVEMENTSTATECOUNT
};
```

每个状态有独立的 LimbPath 配置（`m_Paths[FG/BG][State]`）。

### 6.2 状态转换逻辑

```cpp
// 伪代码：PreControllerUpdate() 中的状态判断
bool moving = controller.IsState(MOVE_LEFT) || controller.IsState(MOVE_RIGHT);
bool prone  = m_ProneState != NOTPRONE;
bool canRun = m_CanRun && controller.IsState(MOVE_FAST);

if (moving) {
    if (prone)   m_MovementState = CRAWL;
    else if (canRun) m_MovementState = RUN;
    else         m_MovementState = WALK;
} else if (controller.IsState(BODY_CROUCH)) {
    m_MovementState = CROUCH;
} else {
    m_MovementState = STAND;
}

// 状态切换时重置步伐计时
if (m_MovementState != oldMoveState) {
    m_StrideStart = true;
}
```

### 6.3 行走时调用 PushAsLimb

```cpp
// 行走/奔跑（WALK 或 RUN 状态）
if (m_MovementState == WALK || m_MovementState == RUN) {
    MovementState pathState = (m_MovementState == RUN) ? RUN : WALK;

    // 读取当前前/背景腿路径进度，用于同步两腿相位
    float FGProgress = m_Paths[FGROUND][pathState].GetRegularProgress();
    float BGProgress = m_Paths[BGROUND][pathState].GetRegularProgress();

    // ── 前景腿 ──
    bool fgRestarted = false;
    bool fgLegActive = m_pFGFootGroup->PushAsLimb(
        m_Pos + RotateOffset(m_pFGLeg->GetParentOffset()),  // 髋关节位置
        m_pFGLeg->GetMaxLength(),                            // 最大腿长
        m_Vel,                                               // 士兵速度
        m_WalkAngle[FGROUND],                                // 路径旋转（地形坡度）
        m_Paths[FGROUND][pathState],                         // 路径数据
        deltaTime,
        &fgRestarted,
        false,                                               // 不影响旋转
        Vector(0.0F, m_Paths[FGROUND][pathState].GetLowestY()),
        pathOffset                                           // 蹲伏偏移
    );

    // ── 背景腿（类似前景腿）──
    bool bgRestarted = false;
    bool bgLegActive = m_pBGFootGroup->PushAsLimb(
        m_Pos + RotateOffset(m_pBGLeg->GetParentOffset()),
        m_pBGLeg->GetMaxLength(),
        m_Vel,
        m_WalkAngle[BGROUND],
        m_Paths[BGROUND][pathState],
        deltaTime,
        &bgRestarted,
        !m_pFGLeg,    // 只有单腿时才影响旋转
        Vector(0.0F, m_Paths[BGROUND][pathState].GetLowestY()),
        pathOffset
    );

    // ── 步伐完成检测 ──
    bool restarted = fgRestarted || bgRestarted;
    if (restarted && !climbing) {
        m_StrideFrame = true;                       // 标记本帧发生了步伐
        RunScriptedFunctionInAppropriateScripts("OnStride");  // 触发 Lua 回调
        // 播放步伐音效（如果配置了 StrideSound）
    }

    // ── 手臂攀爬辅助 ──
    // 如果腿被卡住（PushAsLimb 返回 false），手臂接管攀爬
    if (!fgLegActive) {
        m_ArmClimbing[FGROUND] = true;
        m_pFGHandGroup->PushAsLimb(/* 攀爬路径参数 */);
    }
}
```

### 6.4 爬行时的调用方式

```cpp
else if (m_MovementState == CRAWL && m_ProneState == LAYINGPRONE) {
    // 腿部爬行（使用完整躯体旋转，而非仅地形角度）
    m_pFGFootGroup->PushAsLimb(
        m_Pos + RotateOffset(m_pFGLeg->GetParentOffset()),
        m_pFGLeg->GetMaxLength(),
        m_Vel,
        m_Rotation,             // 注意：这里用完整旋转，而不是 WalkAngle
        m_Paths[FGROUND][CRAWL],
        deltaTime
    );
    m_pBGFootGroup->PushAsLimb(/* 类似 */);

    // 手臂也参与爬行（ARMCRAWL 路径）
    m_pFGHandGroup->PushAsLimb(/* m_Paths[FGROUND][ARMCRAWL] */);
    m_pBGHandGroup->PushAsLimb(/* m_Paths[BGROUND][ARMCRAWL] */);
}
```

### 6.5 站立时保持脚的位置

```cpp
else if (m_MovementState == STAND) {
    // 站立时也调用 PushAsLimb，但使用 STAND 路径（脚保持在原地）
    m_pFGFootGroup->PushAsLimb(
        m_Pos + m_pFGLeg->GetParentOffset(),
        m_pFGLeg->GetMaxLength(),
        m_Vel,
        m_WalkAngle[FGROUND],
        m_Paths[FGROUND][STAND],
        deltaTime,
        nullptr,
        !m_pBGLeg,              // 没有背景腿时影响旋转
        Vector(0.0F, m_Paths[FGROUND][STAND].GetLowestY()),
        pathOffset
    );
}
```

---

## 7. 完整步伐流程（逐步说明）

以下是一个完整步伐周期的执行流程，以"前景腿向前走一步"为例：

### 第 0 步：前置条件

- 士兵处于 WALK 状态
- 前景腿的 LimbPath 处于路径起点（上一步刚完成）
- 背景腿此时处于支撑阶段（路径中途）

### 第 1 步：PreControllerUpdate() 开始

```
1. 读取输入：检测到 MOVE_RIGHT → 状态维持 WALK
2. UpdateWalkAngle(FGROUND)：
     - 从髋关节向下投射光线，检测地面
     - 计算地面坡度角度（例：+10°）
     - m_WalkAngle[FGROUND] 设为 +10° 旋转矩阵
3. UpdateCrouching()：
     - 向上投射光线，检测天花板
     - 头部空间充足 → pathOffset.m_Y = 0（不蹲伏）
4. UpdateLimbPathSpeed()：
     - 当前速度 3 m/s → 速度乘数 0.9
     - 步幅缩放 (1.1, 0.95)
```

### 第 2 步：PushAsLimb() 驱动前景脚

```
5. 计算前景髋关节世界坐标
6. LimbPath 设置：
     - JointPos = 髋关节位置
     - Rotation = +10° 坡度旋转
     - FrameTime = 本帧 deltaTime
7. 循环推进（直到帧时间用完）：
     ├─ 计算当前路径段目标坐标（世界坐标）
     ├─ 获取路径速度向量
     ├─ 调用 PushTravel()：
     │    ├─ Bresenham 推进脚的原子
     │    ├─ 检测地形碰撞
     │    ├─ 碰到地形：计算反作用冲量
     │    └─ 返回冲量向量
     ├─ ReportProgress(脚的当前位置)
     └─ 检查段是否完成，移动到下一段
```

### 第 3 步：力施加到躯体

```
8. 累积本帧所有冲量
9. 过滤反向冲量（X 分量衰减 70%）
10. AddImpulseForce(impulse, rotationOffset)
    → 冲量加入躯体本帧冲量积累器
```

### 第 4 步：Leg 组件更新脚踝（Leg::Update()）

```
11. 接收 AtomGroup 提供的脚踝目标位置
12. UpdateCurrentAnkleOffset()：
      - 平滑插值踝关节偏移到目标
13. UpdateLegRotation()：
      - 计算 NormalizedExtension（当前伸展度）
      - 根据踝关节角度计算腿部旋转
14. 更新精灵帧（代表伸展程度）
15. 更新脚部精灵位置与帧
```

### 第 5 步：物理引擎积分

```
16. Actor::Update() / MovableObject 物理更新：
      - 累积力 → 加速度 → 速度增量
      - m_Vel += (∑forces / mass) × deltaTime
      - m_Pos += m_Vel × deltaTime
17. 重力：m_Vel.m_Y += gravity × deltaTime
18. 旋转弹簧：躯体旋转向目标角度弹回
```

### 第 6 步：路径完成，步伐事件

```
19. 路径所有段完成 → m_Ended = true
20. PushAsLimb 中检测到 restarted = true：
      - m_StrideFrame = true
      - RunScriptedFunctionInAppropriateScripts("OnStride")
      - 播放脚步声
21. RestartFree() 在无地形的新起点重启路径
22. 背景腿进入支撑阶段，前景腿开始新步
```

---

## 8. 地形自适应：坡度与蹲伏

### 8.1 UpdateWalkAngle()：坡度检测

每帧，AHuman 从髋关节向下投射两条光线，根据左右两侧落点的高度差计算地面坡度：

```cpp
void AHuman::UpdateWalkAngle(Layer whichLayer) {
    if (m_Controller.IsState(BODY_JUMP)) {
        // 跳跃中：固定 45° 角，避免腿向下乱晃
        m_WalkAngle[whichLayer] = Matrix(c_QuarterPI * GetFlipFactor());
        return;
    }

    Vector hipPos = m_Pos + RotateOffset(m_pFGLeg->GetParentOffset());

    // 左右各投一条光线，模拟步幅宽度
    Vector hitLeft, hitRight;
    g_SceneMan.CastStrengthRay(hipPos + Vector(-10.0F, 0), Vector(0, rayLen), strength, hitLeft);
    g_SceneMan.CastStrengthRay(hipPos + Vector(+10.0F, 0), Vector(0, rayLen), strength, hitRight);

    // 计算坡度角
    float rawAngle = (hitRight - hitLeft).GetAbsDegAngle();

    // 限制最大坡度（超过 40° 就是墙壁，不走了）
    const float maxAngle = 40.0F;
    float clampedAngle = std::clamp(rawAngle, -maxAngle, maxAngle);

    Matrix walkAngle;
    walkAngle.SetDegAngle(clampedAngle);
    m_WalkAngle[whichLayer] = walkAngle;
}
```

这个旋转角度会传递给 `PushAsLimb()` 的 `rotation` 参数，使整个 LimbPath 跟随地形倾斜，让脚自然地落在斜面上。

### 8.2 UpdateCrouching()：自动蹲伏

每帧从头部向上（和斜前方）投射光线，检测天花板：

```cpp
void AHuman::UpdateCrouching() {
    float desiredYOffset = 0.0F;

    if (m_CrouchAmountOverride != -1.0F) {
        // 脚本强制蹲伏
        desiredYOffset = m_CrouchAmountOverride * m_MaxWalkPathCrouchShift;
    } else if (m_Controller.IsState(BODY_CROUCH)) {
        // 手动蹲伏键
        desiredYOffset = m_MaxWalkPathCrouchShift;
    } else if (!m_Controller.IsState(BODY_JUMP) && m_pHead) {
        // 自动检测天花板
        float desiredHeadRoom = m_SpriteRadius * 2.0F;

        // 从头部向上投射
        Vector hitPos;
        g_SceneMan.CastStrengthRay(m_pHead->GetPos(), Vector(0, -desiredHeadRoom), ..., hitPos);

        // 向前一秒的预测位置也投射（提前蹲伏）
        Vector predictedPos = m_pHead->GetPos() + m_Vel * 1.0F;
        Vector hitPosPredicted;
        g_SceneMan.CastStrengthRay(predictedPos, Vector(0, -desiredHeadRoom), ..., hitPosPredicted);

        // 取两个光线中更低的天花板
        float topY = std::max(hitPos.m_Y, hitPosPredicted.m_Y);
        float headroom = m_pHead->GetPos().m_Y - topY;
        desiredYOffset = desiredHeadRoom - headroom;
    }

    // 平滑过渡（30% 的插值速率）
    float finalOffset = Lerp(..., m_WalkPathOffset.m_Y, desiredYOffset, 0.3F);
    finalOffset = std::clamp(finalOffset, 0.0F, m_MaxWalkPathCrouchShift);

    // 更新蹲伏量 [0=直立, 1=完全蹲伏]
    m_CrouchAmount = finalOffset / m_MaxWalkPathCrouchShift;

    // 写入路径偏移（负 Y = 向上抬起路径）
    m_WalkPathOffset.m_Y = -finalOffset;

    // X 偏移保持身体平衡（跟随速度和头部位置）
    float predictedX = (headPos.m_X - m_Pos.m_X) * 0.15F + m_Vel.m_X;
    m_WalkPathOffset.m_X = predictedX;
}
```

`m_WalkPathOffset` 作为 `positionOffset` 传给 `PushAsLimb()`，使整个 LimbPath 整体向上或向前平移，实现蹲伏时腿部路径的上移。

---

## 9. 动态速度调整

`UpdateLimbPathSpeed()` 每帧根据士兵当前速度动态调整腿部路径的速度和步幅缩放：

```cpp
void AHuman::UpdateLimbPathSpeed() {
    if (m_MovementState == WALK || m_MovementState == RUN || m_MovementState == CRAWL) {

        float speedMultiplier = 1.0F;

        // 蹲伏时减速（由配置的 CrouchWalkSpeedMultiplier 控制）
        if (m_MovementState == WALK) {
            speedMultiplier *= Lerp(0.0F, 1.0F,
                                    1.0F, m_CrouchWalkSpeedMultiplier,
                                    m_CrouchAmount);
        }

        // 低速时减少腿部活动（避免原地快速踏步）
        float maxSpeed = maxLimbPathSpeed * 0.5F * speedMultiplier;
        float velAbs = std::abs(m_Vel.m_X);
        speedMultiplier *= Lerp(0.0F, maxSpeed, minMultiplier, 1.0F, velAbs);

        // 应用速度乘数
        m_Paths[FGROUND][state].SetTravelSpeedMultiplier(speedMultiplier);
        m_Paths[BGROUND][state].SetTravelSpeedMultiplier(speedMultiplier);

        // 根据速度拉伸步幅（快跑时步幅更大）
        //   X 缩放：慢速 0.8 → 快速 1.2
        //   Y 缩放：慢速 0.9 → 快速 1.0
        float strideX = Lerp(0.0F, maxSpeed, 0.8F, 1.2F, velAbs);
        float strideY = Lerp(0.0F, maxSpeed, 0.9F, 1.0F, velAbs);
        m_Paths[FGROUND][state].SetScaleMultiplier(Vector(strideX, strideY));
        m_Paths[BGROUND][state].SetScaleMultiplier(Vector(strideX, strideY));
    }
}
```

这使得步伐自然地随速度变化：慢速时小步，快速时大步，蹲伏时更慢。

---

## 10. 布娃娃物理：不稳定时的肢体

当士兵倒地（`UNSTABLE`）或死亡（`DEAD`/`DYING`），手臂切换为 `FlailAsLimb()` 模式：

```cpp
// 不稳定或死亡时
m_pFGHandGroup->FlailAsLimb(
    m_Pos,
    RotateOffset(m_pFGArm->GetParentOffset()),   // 肩关节偏移
    m_pFGArm->GetMaxLength(),
    m_PrevVel * m_pFGArm->GetJointStiffness(),   // 惯性速度
    m_AngularVel,                                 // 躯体角速度（旋转惯性）
    m_pFGArm->GetMass(),
    deltaTime
);
```

`FlailAsLimb()` 不遵循任何 LimbPath，肢体完全被动地随躯体甩动，产生真实的倒地效果。

---

## 11. 关键数据结构参考

### 运动相关属性（AHuman.h）

| 属性 | 类型 | 说明 |
|------|------|------|
| `m_Paths[2][MOVEMENTSTATECOUNT]` | `LimbPath` | 所有运动状态的肢体路径（前/背景各一套） |
| `m_WalkAngle[2]` | `Matrix` | 前/背景腿的行走角度（地形坡度） |
| `m_WalkPathOffset` | `Vector` | 路径整体偏移（蹲伏/重心调整） |
| `m_CrouchAmount` | `float` | 当前蹲伏量 [0=直立, 1=完全蹲伏] |
| `m_MaxWalkPathCrouchShift` | `float` | 最大蹲伏偏移像素（默认 6.0） |
| `m_StrideFrame` | `bool` | 本帧是否完成了一步（触发 OnStride） |
| `m_StrideStart` | `bool` | 步伐同步标志（状态切换时重置） |
| `m_ArmClimbing[2]` | `bool` | 前/背景手臂是否处于攀爬辅助模式 |
| `m_pFGFootGroup` | `AtomGroup*` | 前景脚碰撞原子组 |
| `m_pBGFootGroup` | `AtomGroup*` | 背景脚碰撞原子组 |

### 物理属性（MovableObject / MOSRotating）

| 属性 | 类型 | 说明 |
|------|------|------|
| `m_Pos` | `Vector` | 世界坐标（每帧由速度积分更新） |
| `m_Vel` | `Vector` | 速度（m/s） |
| `m_AngularVel` | `float` | 旋转速度（rad/s） |
| `m_Mass` | `float` | 质量（kg） |
| `m_Rotation` | `Matrix` | 旋转矩阵 |

---

## 12. 源文件索引

| 文件 | 核心内容 |
|------|---------|
| `Source/Entities/AHuman.h` | 士兵类定义：m_Paths、m_WalkAngle、足部原子组、步伐标志 |
| `Source/Entities/AHuman.cpp` | 主步态逻辑：PreControllerUpdate()、UpdateWalkAngle()、UpdateCrouching()、UpdateLimbPathSpeed() |
| `Source/Entities/Leg.h` | 腿部类定义：踝关节偏移、伸展长度、脚部附件 |
| `Source/Entities/Leg.cpp` | 程序化动画：UpdateCurrentAnkleOffset()、UpdateLegRotation()、UpdateFootFrameAndRotation() |
| `Source/Entities/LimbPath.h` | 路径系统定义：路径点、速度、推力、进度追踪 |
| `Source/Entities/LimbPath.cpp` | 路径计算：GetCurrentVel()、GetNextTimeChunk()、ReportProgress()、GetEffectivePushForce() |
| `Source/Entities/AtomGroup.h` | 碰撞组定义：原子列表、PushAsLimb/FlailAsLimb 签名 |
| `Source/Entities/AtomGroup.cpp` | 物理核心：PushAsLimb()（约1214行）、PushTravel() Bresenham碰撞（约763行） |
| `Source/Entities/Actor.h` | MovementState 枚举定义 |
| `Source/Entities/Actor.cpp` | Actor 基类物理：重力、旋转弹簧等 |

---

## 总结

士兵的移动系统总结为以下三层：

```
【输入层】
  控制器按键（MOVE_LEFT/RIGHT）
       ↓
【路径层】
  LimbPath 路径点 → AtomGroup 推进脚 → 地形碰撞
       ↓ 反作用冲量
【物理层】
  AddImpulseForce() → Vel += F/m × dt → Pos += Vel × dt
       ↑ 视觉层
  Leg 程序化计算旋转与精灵帧（不影响物理）
```

**移动方式：** 通过脚对地面施加力的反作用驱动躯体前进，不是直接修改坐标，也不是简单施加一个前向力，而是通过**脚与地形碰撞的物理反馈**自然涌现的运动。这使得士兵能够自适应各种地形坡度、障碍物和物理情况，无需为每种情况专门编写移动逻辑。
