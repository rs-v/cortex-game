# 地形与地形破坏系统技术文档

## 核心问题：地形与地形破坏是如何实现的？

**结论：地形由三张位图（材质图、前景颜色图、背景颜色图）共同描述。每个像素携带一个材质 ID，材质决定了该像素的物理属性（抗穿透力、密度等）。当子弹或碎片施加的冲量超过材质的结构强度时，该像素被清除为"空气"并转化为飞散的 `MOPixel` 粒子——这就是像素级地形破坏的核心机制。**

---

## 目录

1. [架构总览](#1-架构总览)
2. [三层位图结构](#2-三层位图结构)
3. [材质系统（Material）](#3-材质系统material)
4. [地形生成管线](#4-地形生成管线)
5. [地形破坏：TryPenetrate 穿透检测](#5-地形破坏trypenetrate-穿透检测)
6. [地形破坏：EraseSilhouette 轮廓抹除](#6-地形破坏erasesilhouette-轮廓抹除)
7. [地形破坏：DislodgePixel 像素拆除](#7-地形破坏dislodgepixel-像素拆除)
8. [孤立区域清除：RemoveOrphans](#8-孤立区域清除removeorphans)
9. [像素沉降（Settle）](#9-像素沉降settle)
10. [地形装饰子系统](#10-地形装饰子系统)
11. [关键数据流总结](#11-关键数据流总结)
12. [源文件索引](#12-源文件索引)

---

## 1. 架构总览

```
Scene（场景）
└── SLTerrain（地形层）
    ├── m_MainBitmap          ← 材质图（主位图，每像素存储材质 ID）
    ├── m_FGColorLayer        ← 前景颜色图（玩家所见的可视纹理）
    ├── m_BGColorLayer        ← 背景颜色图（静态背景，不可破坏）
    ├── m_TerrainFrostings[]  ← 地形霜冻（表面覆盖层）
    ├── m_TerrainDebris[]     ← 地形碎屑（随机散布的装饰）
    └── m_TerrainObjects[]    ← 地形对象（预制结构）

SceneMan（场景管理器）
├── GetTerrMatter()           ← 读取指定位置的材质 ID
├── TryPenetrate()            ← 穿透检测与像素破坏
├── DislodgePixel*()          ← 主动拆除像素（工具/脚本用）
└── RemoveOrphans()           ← 孤立地形区域清除

MovableMan（可动物体管理器）
└── Settle Pass               ← 将飞散像素沉降回地形
```

**各组件职责：**

| 组件 | 职责 |
|------|------|
| `SLTerrain` | 存储并渲染地形的三张位图；提供像素读写接口 |
| `Material` | 定义每种材质的物理属性（强度、密度、摩擦、沉降材质等） |
| `Atom` / `AtomGroup` | 驱动运动体与地形的逐像素碰撞检测 |
| `SceneMan::TryPenetrate` | 核心破坏逻辑：判断冲量是否足以击碎地形像素 |
| `MOPixel` | 被击碎的地形像素转化为飞行粒子 |
| `MovableMan` Settle Pass | 将停止运动的粒子重新嵌入地形 |

---

## 2. 三层位图结构

`SLTerrain` 继承自 `SceneLayer`，内部维护**三张 8 位索引位图**，它们共同描述地形状态：

### 2.1 材质图（Material Bitmap）—— `m_MainBitmap`

- 每个像素存储一个 **材质 ID**（0–255）
- `0` = 空气（`g_MaterialAir`），即可通行区域
- 物理系统通过读取此图决定碰撞/穿透行为
- 地形破坏时，将像素改写为 `0`（空气）

```cpp
// 读取指定坐标的材质 ID
int materialID = _getpixel(terrain->GetMaterialBitmap(), posX, posY);

// 破坏：将该像素设置为空气
terrain->SetMaterialPixel(posX, posY, g_MaterialAir);
```

### 2.2 前景颜色图（FG Color Layer）—— `m_FGColorLayer`

- 每个像素存储一个**调色板颜色索引**
- 玩家在屏幕上看到的地形外观就来自这张图
- 地形破坏时，将对应像素设置为透明色（`g_MaskColor`）

```cpp
// 破坏：擦除前景颜色（变透明）
terrain->SetFGColorPixel(posX, posY, g_MaskColor);
```

### 2.3 背景颜色图（BG Color Layer）—— `m_BGColorLayer`

- 渲染在前景之下的静态背景纹理
- **不参与物理计算**，不可被直接破坏
- 前景被破坏后，玩家会看穿前景，露出背景
- 用于判断是否有"背景支撑"：如果某前景像素对应的背景色是透明的，说明该像素处于"悬空"状态，更容易整列崩塌

---

## 3. 材质系统（Material）

每种材质（`Material` 类）是地形物理行为的完整描述，以调色板形式存储（最多 256 种）。

### 3.1 关键属性

| 属性 | 类型 | 含义 |
|------|------|------|
| `m_Index` | `uint8` | 材质 ID（0–255），对应材质图中的像素值 |
| `m_Integrity` | `float` | **结构强度**：击碎该材质像素所需的冲量（kg·m/s），这是地形破坏的核心阈值 |
| `m_Restitution` | `float` | 弹性系数（0=完全塑性，1=完全弹性） |
| `m_Friction` | `float` | 摩擦系数 |
| `m_Stickiness` | `float` | 黏附系数 |
| `m_PixelDensity` | `float` | 像素质量（kg/像素），决定飞散粒子的质量 |
| `m_SettleMaterialIndex` | `uint8` | 像素停止运动后沉降回地形时变为的材质 ID |
| `m_SpawnMaterialIndex` | `uint8` | 像素被击碎时生成的 `MOPixel` 所用的材质 ID（用于特殊效果） |
| `m_IsScrap` | `bool` | 是否为"废料"（来自爆炸碎片），废料更容易被后续撞击崩塌 |
| `m_Priority` | `int` | 显示优先级，值越高越优先渲染 |
| `m_Piling` | `int` | 像素沉降时尝试向上重新定位的次数 |
| `m_UseOwnColor` | `bool` | 若为 true，飞散粒子使用材质自身颜色；否则使用地形像素的实际颜色 |

### 3.2 结构强度与穿透判断

破坏发生的条件（在 `TryPenetrate` 中）：

```cpp
float sqrImpMag = impulse.GetSqrMagnitude();

// 当冲量平方 >= 结构强度平方时，穿透成功
if (sqrImpMag >= (sceneMat->GetIntegrity() * sceneMat->GetIntegrity())) {
    // 执行破坏...
    retardation = -(sceneMat->GetIntegrity() / std::sqrt(sqrImpMag));
}
```

其中 `retardation`（阻滞值）表示穿透时粒子损失的相对速度，结构强度越高，粒子减速越大。

---

## 4. 地形生成管线

每次加载场景时，地形通过以下步骤从材质位图生成可见地形：

```
材质位图（.png，灰度图）
    ↓  SLTerrain::LoadData()
    ↓  SceneLayer::LoadData()       ← 加载材质图到内存
    ↓  TexturizeTerrain()           ← 将材质 ID 映射为颜色纹理
    ↓  TerrainFrosting::FrostTerrain()  ← 应用表面覆盖层
    ↓  TerrainDebris::ScatterOnTerrain()  ← 散布随机碎屑
    ↓  TerrainObject::PlaceOnTerrain()  ← 放置预制结构
    ↓  CleanAir()                   ← 清理空气像素对应的前景色
```

### 4.1 TexturizeTerrain（纹理化）

遍历材质图每个像素，根据材质 ID 查找对应的前景/背景纹理，并平铺写入颜色图：

```cpp
// 伪代码
for each pixel (x, y) in materialBitmap:
    matIndex = getpixel(materialBitmap, x, y)
    material = materialPalette[matIndex]
    
    // 前景颜色图：使用材质的 FG 纹理，或材质固有颜色
    fgColor = material.fgTexture
                ? getpixel(fgTexture, x % fgTexture.w, y % fgTexture.h)
                : material.color
    putpixel(fgColorLayer, x, y, fgColor)
    
    // 背景颜色图：使用材质的 BG 纹理，或场景默认背景纹理
    bgColor = material.bgTexture
                ? getpixel(bgTexture, x % bgTexture.w, y % bgTexture.h)
                : getpixel(defaultBGTexture, x % w, y % h)
    putpixel(bgColorLayer, x, y, bgColor)
```

此过程使用 `std::execution::par_unseq`（C++17 并行算法）多线程处理，显著加快大地图的加载速度。

### 4.2 TerrainFrosting（地形霜冻）

`TerrainFrosting` 在指定目标材质的顶部添加一层"覆盖材质"，例如雪地表面、草地、岩石顶层等。

配置参数：
- `FrostingMaterial`：覆盖层使用的材质
- `TargetMaterial`：覆盖在哪种材质的上方
- `MinThickness` / `MaxThickness`：覆盖层厚度范围（像素）
- `InAirOnly`：是否仅在空气区域覆盖

实现原理：从下到上扫描每列，找到目标材质边界，在其上方的空气像素中绘制覆盖材质。

### 4.3 TerrainDebris（地形碎屑）

`TerrainDebris` 在地形表面随机散布石块、草丛等装饰性精灵。

支持多种放置模式（`DebrisPlacementMode`）：
- `NoPlacementRestrictions`：无限制，可穿透非目标材质向下查找
- `OnSurfaceOnly`：仅放置在表面
- `OnCavitySurfaceOnly`：仅放置在洞穴表面
- `OnOverhangOnly`：仅放置在悬崖顶部

### 4.4 TerrainObject（地形对象）

`TerrainObject` 将预制的复杂结构（如金属箱、洞穴通道等）直接绘制到地形位图中，包含独立的材质图、前景色图和背景色图三个子位图。

---

## 5. 地形破坏：TryPenetrate 穿透检测

`SceneMan::TryPenetrate()` 是地形破坏的**核心入口**，由 `Atom::Travel()` 在每步移动时调用。

### 5.1 调用链

```
MovableObject::Update()
  └── AtomGroup::Travel() / Atom::Travel()
        └── 每步移动检测地形碰撞
              └── SceneMan::TryPenetrate(posX, posY, impulse, velocity, ...)
```

### 5.2 完整破坏流程

```cpp
bool SceneMan::TryPenetrate(posX, posY, impulse, velocity, retardation, airRatio, ...) {
    // 1. 读取该坐标的材质 ID
    unsigned char materialID = getpixel(terrain->GetMaterialBitmap(), posX, posY);
    if (materialID == g_MaterialAir) return true;  // 已经是空气，直接通过
    
    // 2. 获取材质属性
    Material* sceneMat = GetMaterialFromID(materialID);
    float sqrImpMag = impulse.GetSqrMagnitude();
    
    // 3. 判断冲量是否足以穿透
    if (sqrImpMag >= sceneMat->Integrity * sceneMat->Integrity) {
        
        // 4. 生成飞散粒子（MOPixel）
        if (numPenetrations <= 3) {
            Color color = sceneMat->UsesOwnColor()
                         ? sceneMat->GetColor()
                         : terrain->GetFGColorPixel(posX, posY);
            
            MOPixel* pixel = new MOPixel(
                color,
                spawnMat->GetPixelDensity(),
                Vector(posX, posY),
                Vector(随机飞散速度),
                new Atom(...),
                0
            );
            MovableMan.AddParticle(pixel);
        }
        
        // 5. 擦除地形像素
        terrain->SetFGColorPixel(posX, posY, g_MaskColor);
        terrain->SetMaterialPixel(posX, posY, g_MaterialAir);
        
        // 6. 废料崩塌效果：若为废料或无背景支撑，向上崩塌整列
        if (sceneMat->IsScrap() || bgColor[posX][posY] == g_MaskColor) {
            for (testY = posY-1; testY > posY - ScrapCompactingHeight; testY--) {
                // 将上方的废料/悬空像素也转化为飞散粒子
                ...
            }
        }
        
        // 7. 孤立区域清除
        if (removeOrphansRadius > 0 && 随机触发) {
            RemoveOrphans(posX, posY, removeOrphansRadius, removeOrphansMaxArea, true);
        }
        
        // 8. 计算阻滞值（粒子减速）
        retardation = -(sceneMat->GetIntegrity() / sqrt(sqrImpMag));
        return true;
    }
    return false;
}
```

### 5.3 废料崩塌（Scrap Compacting）

当被破坏的像素满足以下任一条件时，触发向上的连锁崩塌：
- **材质为废料**（`m_IsScrap == true`）：爆炸碎片形成的"废料层"更脆弱
- **背景像素为透明**：表示该像素"悬空"，没有背景支撑

系统会向上扫描最多 `m_ScrapCompactingHeight`（默认 25）个像素，将满足条件的像素逐一转化为飞散粒子，模拟崩塌效果。

---

## 6. 地形破坏：EraseSilhouette 轮廓抹除

`SLTerrain::EraseSilhouette()` 用于**整体形状的地形抹除**，例如爆炸波、物体嵌入地形时的碰撞区域等。

### 6.1 调用场景

- 物体爆炸产生碎片（gibs）时
- 移动物体与地形重叠时

### 6.2 工作原理

```
传入精灵 BITMAP（作为"饼干切割模具"）
    ↓  将精灵旋转/缩放渲染到临时位图
    ↓  遍历临时位图的每个非透明像素
    ↓  检查对应地形坐标是否有非空气像素
    ↓  若有：
        ├── 可选：生成 MOPixel（被击碎的地形碎块）
        ├── 将材质图该像素设为 Air
        └── 将前景颜色图该像素设为透明
    ↓  返回所有被抹除的 MOPixel 列表
```

此方法支持旋转（`Matrix rotation`）和缩放（`float scale`），适合处理任意形状的爆炸或碰撞区域。

---

## 7. 地形破坏：DislodgePixel 像素拆除

`SceneMan::DislodgePixel*()` 系列函数提供**主动拆除像素**的接口，主要供 Lua 脚本和工具使用，不依赖物理冲量。

### 7.1 基础函数

```cpp
MOPixel* SceneMan::DislodgePixel(int posX, int posY) {
    // 1. 读取材质
    int materialID = getpixel(terrain->GetMaterialBitmap(), posX, posY);
    if (materialID == g_MaterialAir) return nullptr;
    
    // 2. 创建飞散粒子
    MOPixel* pixel = new MOPixel(颜色, 材质密度, 位置, 零速度, ...);
    MovableMan.AddParticle(pixel);
    
    // 3. 擦除地形像素
    terrain->SetFGColorPixel(posX, posY, g_MaskColor);
    terrain->SetMaterialPixel(posX, posY, g_MaterialAir);
    
    return pixel;
}
```

### 7.2 扩展形状变体

| 函数 | 形状 | 参数 |
|------|------|------|
| `DislodgePixel(x, y)` | 单点 | 坐标 |
| `DislodgePixelBool(x, y, delete)` | 单点 | 坐标 + 是否立即标记删除 |
| `DislodgePixelCircle(centre, radius, delete)` | 圆形 | 圆心、半径 |
| `DislodgePixelRing(centre, r1, r2, delete)` | 圆环 | 圆心、内径、外径 |
| `DislodgePixelBox(ul, lr, delete)` | 矩形 | 左上角、右下角 |
| `DislodgePixelLine(start, ray, skip, delete)` | 线段 | 起点、方向向量、跳过间隔 |

### 7.3 Lua 脚本示例

```lua
-- 在 Lua 中使用 DislodgePixel 实现挖掘工具
local px = SceneMan:DislodgePixel(checkPos.X, checkPos.Y)
if px then
    local material = SceneMan:GetMaterialFromID(terrCheck)
    -- 根据材质强度决定是否能挖掘
    if material.StructuralIntegrity <= self.digStrength then
        -- 挖掘成功...
    end
end
```

---

## 8. 孤立区域清除：RemoveOrphans

当地形像素被破坏后，可能留下与主体地形失去连接的**浮岛碎块**。`RemoveOrphans()` 通过**洪水填充算法**检测并清除这些孤立区域。

### 8.1 工作原理

```
以破坏点为中心，设定搜索半径 radius 和最大面积 maxArea
    ↓  第一次洪水填充（仅计算面积，不删除）
        ├── 若填充在半径边界仍有非空气像素 → 连接到主体地形 → 不是孤岛 → 中止
        └── 若填充完全在半径内 → 记录总面积
    ↓  若总面积 ≤ maxArea（孤岛足够小）
        └── 第二次洪水填充（实际删除）
              ├── 每个孤岛像素生成 MOPixel 飞散粒子
              └── 将材质图和颜色图对应像素清空为空气
```

### 8.2 关键参数

| 参数 | 含义 |
|------|------|
| `radius` | 搜索半径（最大 `MAXORPHANRADIUS`） |
| `maxArea` | 触发清除的最大孤岛面积（像素数） |
| `remove` | `true` = 实际删除；`false` = 仅计算面积 |

### 8.3 触发时机

在 `TryPenetrate()` 中，当传入了有效的 `removeOrphansRadius` 和 `removeOrphansMaxArea` 时，按 `removeOrphansRate` 概率触发：

```cpp
if (removeOrphansRadius && removeOrphansMaxArea && 
    removeOrphansRate > 0 && RandomNum() < removeOrphansRate) {
    RemoveOrphans(posX, posY, removeOrphansRadius, removeOrphansMaxArea, true);
}
```

---

## 9. 像素沉降（Settle）

被击碎的地形像素以 `MOPixel` 形式在场景中飞行，最终会重新沉降回地形。

### 9.1 沉降触发条件

当 `MOPixel` 的速度足够小（由 `AtomGroup::Travel()` 判定）时，`MovableObject::m_ToSettle` 被设为 `true`。

### 9.2 沉降处理流程（MovableMan::Update()）

```cpp
// Settle Pass（在所有 Update 和删除之后执行）
if (m_SettlingEnabled) {
    for each particle marked ToSettle:
        Vector pos = particle->GetPos();
        Material* terrMat = GetTerrainMaterialAt(pos);
        int piling = particle->GetMaterial()->GetPiling();
        
        // Piling：尝试向上移动，避免与相同材质重叠
        if (piling > 0) {
            for s in 0..piling:
                if 当前位置的地形材质 == 粒子材质:
                    if s 为偶数: pos.Y -= 1   // 向上移动
                    else:        pos.X ±= 1   // 横向移动
        }
        
        // 若粒子绘制优先级高于当前地形材质的优先级，才能覆盖
        if particle.DrawPriority >= terrMat.Priority:
            particle->DrawToTerrain(terrain)  // 写入地形位图
        
        delete particle  // 粒子消失
}
```

### 9.3 沉降材质转换

当 `m_SettleMaterialIndex != 0` 时，像素沉降时会转换为另一种材质（例如液态岩浆凝固后变成石头）。

---

## 10. 地形装饰子系统

以下子系统负责在场景加载时为地形添加视觉细节，它们只修改颜色图和材质图，不参与物理计算（除非被后续的破坏逻辑处理）。

### 10.1 TerrainFrosting（表面覆盖）

在目标材质顶部生长一层覆盖材质，常用于：
- 雪地：在石头上方添加雪材质
- 草地：在泥土上方添加草材质
- 湿润表面等

```
配置示例（.ini 文件）：
AddTerrainFrosting = TerrainFrosting
    FrostingMaterial = Snow
    TargetMaterial   = Rock
    MinThickness     = 3
    MaxThickness     = 8
    InAirOnly        = 1
```

### 10.2 TerrainDebris（随机碎屑）

在地形表面或指定位置随机散布小精灵，例如石块、植被、裂缝等。支持多种放置模式、旋转角度范围和密度控制。

### 10.3 TerrainObject（预制结构）

将设计好的多层位图结构"印刷"到地形中，适合放置洞穴入口、金属板、机械装置等复杂场景元素。

---

## 11. 关键数据流总结

### 地形生成数据流

```
磁盘文件（材质位图 .png）
    → SceneLayer::LoadData()           // 加载原始材质图
    → TexturizeTerrain()               // 材质 ID → 颜色纹理（并行）
    → TerrainFrosting::FrostTerrain()  // 绘制表面覆盖层
    → TerrainDebris::ScatterOnTerrain()// 随机放置碎屑精灵
    → TerrainObject::PlaceOnTerrain()  // 印刷预制结构
    → CleanAir()                       // 清理空气像素的前景色
    → 地形就绪，渲染前景颜色图
```

### 地形破坏数据流（弹药/碎片路径）

```
MovableObject::Update()
    → AtomGroup::Travel() 或 Atom::Travel()
        → 每步移动时：检测地形碰撞（读取材质图）
        → SceneMan::TryPenetrate()
            ├── 判断：冲量² ≥ 结构强度²？
            │   是 →  生成 MOPixel 飞散粒子（使用材质颜色/密度）
            │          擦除材质图该像素 → Air
            │          擦除前景颜色图该像素 → 透明
            │          废料崩塌（若条件满足）
            │          RemoveOrphans（可选，随机触发）
            │          返回 retardation（减速值）→ 粒子继续但减速
            └── 否 →  返回 false → 粒子反弹（根据 Restitution）
    → 被击碎的 MOPixel 加入 MovableMan 的粒子列表
    → 粒子飞行、受重力影响
    → 粒子停止时：Settle Pass
        → DrawToTerrain()  // 重新写入地形（沉降）
        → 粒子销毁
```

### 工具/脚本挖掘数据流

```
Lua 脚本
    → SceneMan:DislodgePixel(x, y)
        ├── 创建 MOPixel（零速度）
        ├── 擦除材质图 → Air
        └── 擦除前景颜色图 → 透明
    → MOPixel 在世界中自由运动（受重力影响）
    → 最终沉降或超出生命周期后销毁
```

---

## 12. 源文件索引

| 文件 | 职责 |
|------|------|
| `Source/Entities/SLTerrain.h/.cpp` | 地形主类：三层位图管理、地形生成、EraseSilhouette |
| `Source/Entities/Material.h/.cpp` | 材质属性定义（强度、密度、颜色、沉降等） |
| `Source/Entities/TerrainFrosting.h/.cpp` | 地形表面覆盖层 |
| `Source/Entities/TerrainDebris.h/.cpp` | 随机碎屑散布 |
| `Source/Entities/TerrainObject.h/.cpp` | 预制地形结构 |
| `Source/Managers/SceneMan.h/.cpp` | TryPenetrate、DislodgePixel*、RemoveOrphans |
| `Source/System/Atom.h/.cpp` | 单原子碰撞/旅行逻辑，调用 TryPenetrate |
| `Source/Entities/AtomGroup.h/.cpp` | 多原子组合，驱动运动体与地形的全面碰撞 |
| `Source/Entities/MOPixel.h/.cpp` | 飞散粒子实体（被击碎的地形像素） |
| `Source/Managers/MovableMan.h/.cpp` | 粒子管理、Settle Pass（沉降处理） |
| `Source/Entities/SceneLayer.h/.cpp` | 位图层基类（SLTerrain 的父类） |
