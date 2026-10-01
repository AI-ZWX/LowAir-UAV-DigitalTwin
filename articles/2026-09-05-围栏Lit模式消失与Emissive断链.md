# 围栏在 Lit 渲染模式下消失：一条被 Python 静默改断的 Emissive 链

> 现象一句话：视口 Unlit 模式下围栏颜色完美，一切到 Lit 游戏渲染，围栏整个消失——不是变黑，是透明到不存在。

## 现象

2026-09-05，场景视觉调整。当时的目标很单纯：给电子围栏的材质（M_FenceYellow / M_FenceRed）加一点边缘发光效果，让它在低空场景里更醒目。

用 Python 脚本做了三件事：改 UNLIT、加 Fresnel 节点、加 EdgeBoost 节点。脚本跑完，日志确认 `ShadingModel=0 (UNLIT)` 生效，材质也落盘了。

然后开 PIE 一看：Unlit 视口下围栏 A 紫、围栏 B 青绿、地标光柱全都正常；**切换到 Lit 游戏渲染，围栏全部消失**。

## 排查路径

**第一猜：UNLIT 没生效。** 查日志，`ShadingModel=0 (UNLIT)` 白纸黑字。推翻。

**第二猜：颜色链路没接上。** 用 `get_material_property_input_node` 直接查材质的 Emissive 输入端——**Emissive input node = NONE，悬空的**。输出端什么都没接，UNLIT 材质的发光值就是 0，等于全透明。Lit 模式下材质是否可见基本靠光照和自发光贡献，自发光为 0、又没有正常的光照链路，就什么都不剩。

**第三猜：什么时候断的？** 回看我自己的脚本，STEP5 加 Fresnel 的那一步：要把原 Emissive 链拆开、插入 Fresnel、再重连。拆开和重连用的是 `MaterialEditingLibrary.connect_material_expressions`。

把这一步单独拿出来实测：这个 API 在我这个环境里**一律返回 False**——不是偶尔失败，是全部失败，而且不抛异常、不报警告，安安静静地返回 False。另一个材质 M_SceneDarken 上四条连接全 False，是同样的实锤。

串起来就是完整的事故链：

```
原 Emissive 链被拆开 → 重连 API 返回 False → 链没接回去
→ Emissive 输出悬空 = 0 → UNLIT 发光值 = 0 → 围栏全透明 → "消失"
```

## 根因

**UNLIT 化本身没错，错在 Fresnel 修改把 Emissive 链弄断了。**

更深一层：Python 的 MaterialEditingLibrary 连接类 API 在我的环境里不可靠——返回 False 不报错，是静默失败。脚本以为自己改完了材质，实际只完成了一半，而日志里什么都看不出来。

## 修复

回滚 M_FenceYellow / M_FenceRed 的 Fresnel 修改，恢复原始 Emissive 直连链，保留 UNLIT。回滚只能在编辑器里手动核对着做——git 不跟踪 .uasset 二进制资产，没法直接 `git checkout` 回滚。09-06 确认解决。

## 下回怎么做

- [ ] **禁止再用 Python 改材质节点**——创建和修改材质只走三条路：编辑器手动、C++、蓝图
- [ ] **返回布尔的 API，一律打印返回值复核**，不信"没报错就是成功了"
- [ ] **材质修改必须三重验证落盘**：`save_asset` 返回值不可信，要用 `OBJ SAVEPACKAGE` 命令 + 磁盘文件时间戳 + `get_material_property_input_node` 查连接，三样都对才算改完
- [ ] **改材质前先留原状**（截图或导出），.uasset 不进 git，改坏了没有后悔药

## 素材出处（均可复验）

- `F:\低空数字孪生归拢系统\11-技术实现笔记\场景问题与方案\场景视觉问题台账_20260905.md`（问题 2 完整记录 + 核心教训 1-2）
- `F:\低空数字孪生归拢系统\00-项目总览\每日记录\项目进度归档_2026-09-06.md`（"Fresnel 修改回滚，Emissive 链恢复"解决记录）
- `F:\低空数字孪生归拢系统\04-踩坑实录\2026-09-04-UE5.7Python远程验证API偏差.md`（UE5.7 Python API 四条偏差的独立佐证）
