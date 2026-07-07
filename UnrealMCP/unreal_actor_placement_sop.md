# 虚幻引擎 Actor 精准贴地摆放 SOP (Standard Operating Procedure)

在虚幻引擎关卡编辑中，将新拖入或生成的 Actor（物体、角色蓝图等）完美贴合在复杂地形（Landscape）或静态网格体（Static Mesh）的碰撞表面上，是经常遇到的需求。本文记录了手动操作与自动化脚本两种可复用的对齐方案，并列出关键避坑指南。

---

## 方案一：编辑器内置常规操作 (适用于手动摆放)

虚幻编辑器提供了快速贴合地面的快捷操作。

### 操作步骤
1. 打开 Content Browser，将一个或多个 Actor **拖入关卡中**。
2. 保持选中这些 Actor。
3. 按下键盘上的 **`End` 键**（部分紧凑键盘需按 `Fn + Right` 或对应组合键）。
4. **效果**：Actor 会瞬间垂直下落，精准贴合在它们下方的碰撞地表上。

### ⚠️ 关键避坑与原理
*   **向下射线 Sweep 原理**：`End` 键（后台对应控制台命令 `SetActorToFloor`）的底层原理是从 Actor 当前的 **Pivot（轴心点）** 出发，**垂直向下**发射一条碰撞检测射线。
*   **地底失效陷阱（重要）**：如果 Actor 当前已经被埋在地面下方（例如地表 Z = 198.0，而 Actor 被生成在了 Z = 100.0 或更低处），此时向下发射射线将永远检测不到上方的路面，导致 `End` 键失效，Actor 依然被埋在地下。
*   **解决方案**：如果贴地失效，**请先使用平移工具将 Actor 往上拉到半空中（确保高于地表），然后再按下 `End` 键**，此时它们便能完美下落贴地。

---

## 方案二：Python 脚本自动化 (适用于批量生成与程序化对齐)

当通过 Python 脚本或自动化工具（如 MCP）批量生成角色并一字排开时，由于摄像机仰俯角和虚幻 Python API 自身的转换限制，容易出现“高低倾斜”或“属性不生效”的 Bug。以下是经过验证的终极自动化贴地方案。

### 核心 Python 脚本模板

```python
import unreal

def snap_heroes_to_floor():
    all_actors = unreal.EditorLevelLibrary.get_all_level_actors()
    # 1. 过滤需要对齐的目标 Actor（以 Hero_ 前缀为例）
    heroes = [a for a in all_actors if "Hero_" in a.get_actor_label()]
    world = unreal.EditorLevelLibrary.get_editor_world()
    
    # 2. 准备可见性碰撞通道 (Visibility Channel)
    trace_channel = unreal.TraceTypeQuery.TRACE_TYPE_QUERY1
    draw_debug = unreal.DrawDebugTrace.NONE
    
    aligned_count = 0
    for hero in heroes:
        loc = hero.get_actor_location()
        
        # 3. 构造射线：从 Actor 当前 XY 坐标上方高空 (Z=1000) 垂直投射到地底 (Z=-1000)
        # 即使小人原先在地下，从 1000 往下投射也绝对能打中路面上表面
        start = unreal.Vector(loc.x, loc.y, 1000.0)
        end = unreal.Vector(loc.x, loc.y, -1000.0)
        
        # 4. 执行单线射线检测 (Editor Line Trace)
        hit_result = unreal.SystemLibrary.line_trace_single(
            world,
            start,
            end,
            trace_channel,
            False,        # trace_complex
            [hero],       # ignore_actors
            draw_debug,   # 必须是 unreal.DrawDebugTrace 枚举，不能传整型或字符串
            True,         # ignore_self
            unreal.LinearColor(1, 0, 0, 1),
            unreal.LinearColor(0, 1, 0, 1)
        )
        
        # 5. 解包 HitResult。虚幻 Python 的 HitResult 结构体需要通过 to_tuple() 获取数据
        tup = hit_result.to_tuple()
        # tup[0] (bool) 代表 bBlockingHit（是否碰撞击中）
        # tup[4] (unreal.Vector) 代表 Location（击中地表的精确3D坐标）
        if tup[0] and tup[4] is not None:
            hit_loc = tup[4]
            
            # 6. 【避坑重点】必须显式重新构造 Vector 对象！
            # 直接赋值如 loc.z = hit_loc.z 会因为虚幻 Python Wrapper 的局部拷贝机制而导致修改失效。
            new_loc = unreal.Vector(loc.x, loc.y, hit_loc.z)
            hero.set_actor_location(new_loc, False, False)
            
            print(f"ALIGN_SUCCESS: Snapped '{hero.get_actor_label()}' to ground Z = {hit_loc.z:.2f}")
            aligned_count += 1
        else:
            # 备用方案 Z 轴高度
            fallback_loc = unreal.Vector(loc.x, loc.y, 200.0)
            hero.set_actor_location(fallback_loc, False, False)
            print(f"ALIGN_MISS: No ground hit. Defaulted to Z = 200.0")

    print(f"COMPLETED: Snapped {aligned_count} actors to ground surface.")

snap_heroes_to_floor()
```

### ⚠️ 自动化开发核心避坑指南

1.  **右向量 Z 轴归一化（防止横排倾斜）**：
    如果要以相机视口为基准横向一字排开物体，直接使用 `right` 向量会导致 Z 轴偏移（因为相机有 Pitch 俯仰角，右向量会带斜率）。
    *   **正规化方法**：必须将右向量投影到水平面并重新归一化：
        `horizontal_right = unreal.Vector(right.x, right.y, 0.0).normal()`
2.  **强制构造 `unreal.Vector`**：
    虚幻 Python 里的 `Vector`、`Rotator` 在执行诸如 `loc.z = 100` 或 `rot.pitch = 0` 的属性修改时，往往只改动了 Python 的临时拷贝，**不会同步写入虚幻 C++ 内存**。
    *   **避坑方法**：必须使用构造函数整体重构：
        `spawn_loc = unreal.Vector(x, y, ground_z)`
        `spawn_rot = unreal.Rotator(pitch=0.0, yaw=yaw, roll=0.0)`
3.  **指定 Rotator 键值对参数**：
    在 `unreal.Rotator()` 实例化时，如果不带参数名，容易因为 positional arguments 的底层映射失误把 `yaw` 值套入 `roll` 或 `pitch`，导致生成的 Actor 倒扣在地上。
    *   **避坑方法**：实例化时务必显式指定参数名：
        `unreal.Rotator(pitch=0.0, yaw=yaw_value, roll=0.0)`
4.  **`unreal.HitResult` 的属性提取**：
    虚幻 Python 不允许对返回 of `HitResult` 进行 tuple 解包，且由于结构体序列化包装的原因，不能直接访问 `.actor` 属性。
    *   **解包方法**：应通过 `tup = hit_result.to_tuple()` 转换成元组，通过 `tup[0]` 检查是否击中，`tup[4]` 获取地表绝对坐标。
