# 虚幻编辑器快捷键与控制台命令映射指南 (Editor Command Mapping)

在虚幻引擎自动化流（如 Python 脚本、编辑器公用工具或远程 MCP 控制）中，**直接调用编辑器控制台指令 (Console Command)** 是代替模拟物理键盘输入（如按 `End`、`Ctrl+End` 等）的最佳实践。

这种模式不需要虚幻窗口置顶或获得焦点，能够在后台同步执行，100% 稳定。

---

## 常用编辑器快捷键与命令映射表

在关卡编辑器中，如果选中了某些 Actor，可以通过控制台或脚本执行以下命令：

| 物理快捷键 | 对应控制台命令 (Console Command) | 功能说明 |
| :--- | :--- | :--- |
| **`End`** | **`SetActorToFloor`** | 将选中的 Actor 垂直下落，对齐并贴合在其下方的碰撞面上（Sweep Down）。 |
| **`Shift + End`** | **`SnapBrushToFloor`** | 将选中的 Brush/Actor 的底部边框精确对齐到下方表面。 |
| **`Ctrl + End`** | **`SnapOriginToGrid`** | 将选中的 Actor 的 Pivot（轴心原点）自动对齐到当前编辑器配置的网格点上。 |
| **`Alt + End`** | **`SnapPivotToFloor`** | 将选中的 Actor 的轴心位置重置并吸附到当前下落点高度。 |
| **`Alt + Ctrl + End`**| **`SnapPivotToGrid`** | 将选中 Actor 的轴心点对齐到网格。 |
| **`方向键` (平移移动)** | **`MoveActor`** *(带参数)* | 绕特定轴移动当前选中的 Actor。例如：<br> - `MoveActor X=100` *(X轴向前移1米)*<br> - `MoveActor Y=-50` *(Y轴向左移半米)*<br> - `MoveActor X=100 Y=100 Z=0` *(横向对角平移)* |

---

## 在 Python 脚本中的调用方式

在编辑器 Python 环境中，通过 `unreal.SystemLibrary.execute_console_command` 并传入编辑器 World 实例即可同步运行：

```python
import unreal

# 1. 选中您想要操作的 Actor (若未选中，执行命令会静默跳过)
# 比如选中所有带有 "Hero_" 前缀的 Actor
all_actors = unreal.EditorLevelLibrary.get_all_level_actors()
target_actors = [a for a in all_actors if "Hero_" in a.get_actor_label()]
unreal.EditorLevelLibrary.set_selected_level_actors(target_actors)

# 2. 获取当前的编辑器 World
editor_world = unreal.EditorLevelLibrary.get_editor_world()

# 3. 模拟按下 'End' 键贴合地面
unreal.SystemLibrary.execute_console_command(editor_world, "SetActorToFloor")

# 4. 模拟按下 'Ctrl + End' 对齐到网格
unreal.SystemLibrary.execute_console_command(editor_world, "SnapOriginToGrid")

# 5. 模拟按下 '方向键' 在 Y 轴上向右侧移动 1.5 米 (150厘米)
unreal.SystemLibrary.execute_console_command(editor_world, "MoveActor Y=150")

# 6. 清除选中状态 (可选)
unreal.EditorLevelLibrary.set_selected_level_actors([])
```
