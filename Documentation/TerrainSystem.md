# Terrain & Terrain Destruction System

## Overview / 概述

The terrain in Cortex Command is a **pixel-based, fully destructible physical terrain** represented by three bitmaps: a material bitmap, a foreground color bitmap, and a background color bitmap. Each pixel carries a material ID that determines its physical properties (structural integrity, density, friction, etc.). When a projectile or debris applies enough impulse to exceed a material's structural integrity threshold, the pixel is cleared to "air" and converted into a flying `MOPixel` particle—this is the core of the pixel-level terrain destruction mechanism.

地形由三张位图描述：材质图、前景颜色图、背景颜色图。每个像素携带一个材质 ID，决定其物理属性。当施加冲量超过材质结构强度时，像素被清为"空气"并转化为 `MOPixel` 粒子——这是像素级地形破坏的核心机制。

## Three-Layer Bitmap Structure / 三层位图结构

`SLTerrain` inherits from `SceneLayer` and maintains three 8-bit indexed bitmaps:

`SLTerrain` 继承自 `SceneLayer`，维护三张 8 位索引位图：

| Layer | Purpose |
|-------|---------|
| **Material Bitmap** (`m_MainBitmap`) | Each pixel stores a material ID (0–255). Index 0 = Air. This drives all physics. |
| **FG Color Layer** (`m_FGColorLayer`) | The visible foreground texture the player sees. Set to `g_MaskColor` (transparent) when a pixel is destroyed. |
| **BG Color Layer** (`m_BGColorLayer`) | The static background texture rendered behind the foreground. Not directly destructible. Used to detect "unsupported" pixels during destruction. |

## Material System / 材质系统

Each material (`Material` class) fully describes the physical behavior of one type of terrain pixel:

每种材质（`Material` 类）完整描述一类地形像素的物理行为：

| Property | Type | Meaning |
|----------|------|---------|
| `m_Index` | `uint8` | Material ID (0–255), matches pixel value in the material bitmap |
| `m_Integrity` | `float` | **Structural integrity**: impulse (kg·m/s) needed to destroy this pixel |
| `m_Restitution` | `float` | Elasticity (0 = plastic, 1 = perfectly elastic) |
| `m_Friction` | `float` | Friction coefficient |
| `m_PixelDensity` | `float` | Pixel mass (kg/pixel), used as mass for the spawned `MOPixel` |
| `m_SettleMaterialIndex` | `uint8` | Material this becomes when settling back into terrain |
| `m_SpawnMaterialIndex` | `uint8` | Material spawned as debris particles for special effects |
| `m_IsScrap` | `bool` | Scrap material (from gibs/explosions) collapses more easily |
| `m_Piling` | `int` | Times a settling pixel tries to reposition upward to avoid stacking on same material |

## Terrain Generation Pipeline / 地形生成管线

```
Material Bitmap (.png, on disk)
  → SceneLayer::LoadData()              Load raw material bitmap
  → TexturizeTerrain()                  Map material IDs → color textures (parallel)
  → TerrainFrosting::FrostTerrain()     Paint surface cover layers (e.g. snow, grass)
  → TerrainDebris::ScatterOnTerrain()   Randomly scatter decorative debris sprites
  → TerrainObject::PlaceOnTerrain()     Stamp prefabricated terrain structures
  → CleanAir()                          Clear foreground color of all air pixels
  → Terrain ready for rendering
```

`TexturizeTerrain()` is parallelized with `std::execution::par_unseq` (C++17 parallel algorithms).

## Terrain Destruction: TryPenetrate / 地形破坏：穿透检测

`SceneMan::TryPenetrate()` is the **core entry point** for terrain destruction. It is called by `Atom::Travel()` on every step an atom takes through the scene.

`SceneMan::TryPenetrate()` 是地形破坏的核心入口，由 `Atom::Travel()` 在每步移动时调用。

### Logic Flow / 执行逻辑

```
1. Read material ID from the terrain material bitmap at (posX, posY)
   If materialID == Air → already destroyed, return true

2. Check: impulse² ≥ integrity²?
   If NO → return false (projectile bounces)

3. If YES (penetration occurs):
   a. Determine spawn material and color (material's own color, or terrain pixel's color)
   b. Spawn MOPixel flying particle (velocity ∝ incoming velocity × spray scale)
   c. Erase terrain pixel: materialBitmap[x,y] = Air, fgColorLayer[x,y] = MaskColor
   d. Compute retardation = -(integrity / √impulse) applied to the penetrating atom

4. Scrap compaction: if pixel is scrap OR has no BG pixel (unsupported):
   Scan upward up to ScrapCompactingHeight pixels; convert each qualifying
   pixel above into MOPixel particles (column collapse effect)

5. Orphan removal (probabilistic): call RemoveOrphans() to detect and
   remove floating terrain islands created by the penetration
```

### Call Chain / 调用链

```
MovableObject::Update()
  └── AtomGroup::Travel()
        └── Atom::Travel()
              └── SceneMan::TryPenetrate()
```

## Terrain Destruction: EraseSilhouette / 地形破坏：轮廓抹除

`SLTerrain::EraseSilhouette()` erases an arbitrary sprite-shaped region from the terrain, used for explosions and gib impacts.

`SLTerrain::EraseSilhouette()` 将任意精灵形状从地形中抹除，用于爆炸和碎片冲击。

```
Given: sprite BITMAP, position, pivot, rotation, scale
1. Render sprite (rotated/scaled) into a temporary bitmap
2. For each non-mask pixel in the temp bitmap:
   - Find corresponding terrain coordinates
   - If terrain pixel is non-air:
     * Optionally: create MOPixel (dislodged terrain debris)
     * Clear material pixel → Air
     * Clear FG color pixel → MaskColor
3. Return deque of all dislodged MOPixels (caller takes ownership)
```

## Terrain Destruction: DislodgePixel / 地形破坏：像素拆除

`SceneMan::DislodgePixel*()` provides **direct pixel removal** without physics — used by Lua scripts and tools like the Constructor.

`SceneMan::DislodgePixel*()` 提供不依赖物理冲量的主动像素拆除接口，供 Lua 脚本和工具使用。

| Function | Shape |
|----------|-------|
| `DislodgePixel(x, y)` | Single pixel |
| `DislodgePixelBool(x, y, delete)` | Single pixel + mark for deletion |
| `DislodgePixelCircle(centre, radius, delete)` | Filled circle |
| `DislodgePixelRing(centre, r1, r2, delete)` | Ring (donut) |
| `DislodgePixelBox(ul, lr, delete)` | Rectangle |
| `DislodgePixelLine(start, ray, skip, delete)` | Line segment |

## Orphan Region Removal / 孤立区域清除

After pixel removal, isolated floating terrain islands can be left behind. `RemoveOrphans()` uses **flood-fill** to detect and remove them.

像素被移除后，可能留下孤立的浮岛。`RemoveOrphans()` 使用洪水填充算法检测并清除它们。

```
1. Flood-fill from destruction point within radius (no removal yet)
   - If fill reaches the radius boundary with solid pixels → connected to main terrain → skip
   - If fill is fully contained → it's an orphaned island

2. If island area ≤ maxArea:
   - Second flood-fill pass: convert each orphan pixel to an MOPixel particle
   - Clear material and color bitmaps
```

## Pixel Settling / 像素沉降

When `MOPixel` particles come to rest, they settle back into the terrain.

`MOPixel` 粒子静止后沉降回地形。

```
MovableMan::Update() — Settle Pass:
  For each particle marked ToSettle:
    1. Apply piling: if landing on same material, try to relocate upward/sideways
    2. If particle.DrawPriority ≥ terrain.Priority at landing position:
         DrawToTerrain() — write pixel back into material and color bitmaps
    3. Delete the particle
```

## Source Files / 源文件索引

| File | Role |
|------|------|
| `Source/Entities/SLTerrain.h/.cpp` | Terrain class: three-layer bitmap management, terrain generation, EraseSilhouette |
| `Source/Entities/Material.h/.cpp` | Material property definitions (integrity, density, color, settling, etc.) |
| `Source/Entities/TerrainFrosting.h/.cpp` | Surface cover layers (snow, grass, etc.) |
| `Source/Entities/TerrainDebris.h/.cpp` | Random debris scattering |
| `Source/Entities/TerrainObject.h/.cpp` | Prefabricated terrain structures |
| `Source/Managers/SceneMan.h/.cpp` | TryPenetrate, DislodgePixel*, RemoveOrphans |
| `Source/System/Atom.h/.cpp` | Single-atom travel and collision, calls TryPenetrate |
| `Source/Entities/AtomGroup.h/.cpp` | Multi-atom group, drives full-body terrain collision |
| `Source/Entities/MOPixel.h/.cpp` | Flying pixel entity (dislodged terrain pixel) |
| `Source/Managers/MovableMan.h/.cpp` | Particle management, Settle Pass |
| `Source/Entities/SceneLayer.h/.cpp` | Bitmap layer base class (parent of SLTerrain) |
