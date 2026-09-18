# xiaoMing 固件重构设计

日期：2026-08-10
状态：已获用户批准

> ⚠️ **历史文档**：本文记录 2026-08-10 当次重构的当时状态。此后串口协议已迁移到
> [HOPE-Remote](https://github.com/ZhangKeLiang0627/HOPE-Remote)：红外与 433 合并到同一个
> Serial1（115200 8N1），槽号区间区分介质，`RmCmd` 的 `port` 字段已被 `slot` + `learn` 取代。
> 文中涉及 `port` / `Serial1` / `Serial2` 的描述与当前代码不一致，**以 [README](../../../README.md) 为准**。

## 背景

`Firmware/xiaoming.hd` 是 ASRPRO 语音灯箱固件（pro-code 单文件，约 720 行）。当前核心 `ASR_CODE()` 是一个约 90 个 case 的巨型 `switch`，存在大量重复：

- 颜色设置重复约 20 次（`setBrightness + pixel_set_all_color + pixel_show`）
- 红外/433 指令重复约 44 次（`SerialN.write("xxNN"/"fsNN")`）

重构目标：可读性、可维护性（新增 / 删除 / 修改指令容易）、优雅。行为 100% 保持（除一处亮度回绕 bug，见「Bug 修复」）。

## 约束

- 保持 ASRPRO pro-code 单文件格式：保留 includes、`setup()` 中 `{ID:xx,...}` 语音定义注释、`/** edittype="asr_procode" */` 尾部标记、标准入口（`setup` / `ASR_CODE` / `hardware_init`）。
- 用户直接编辑代码，IDE 不会重新生成 `ASR_CODE` 内部。
- 本地无 ASRPRO 编译环境，由用户在 IDE 中编译实测。

## 设计

### 数据驱动查表

两张静态表 + 一个特殊逻辑 handler，`ASR_CODE` 三步分发。

### 颜色表 `COLOR_TABLE`（16 行）

结构体：`{ snid, r, g, b, brightness, fixedBrightness, applyBrightness }`

- `fixedBrightness`：true=用固定亮度值；false=用当前 `brightNess`
- `applyBrightness`：是否调用 `setBrightness`（关灯为 false，严格保真）

覆盖 snid：1,2,3,4,5,6,7,8,9,10,11,12,33,34,35,36（多色灯光 + 场景）

### 红外/433 指令表 `RM_TABLE`（60 行）

结构体：`{ snid, port, cmd }`

- `port`：1=Serial1（红外），2=Serial2（433）
- `cmd`：4 字节 ASCII，如 `"xx22"` / `"fs26"`

覆盖 snid：37–80（红外，44 行）、161–176（433，16 行）

### ASR_CODE 分发

```
set_state_enter_wakeup(20000);
1. 颜色表命中 → applyColor() 并 return
2. 指令表命中 → sendRm() 并 return
3. handleSpecials(snid)  ← 小 switch：音量5组、亮度5组、彩虹2、睡眠+定时2、回家/出门2
```

### 辅助函数

- `applyColor(const ColorCmd&)` — 处理 `fixedBrightness` / `applyBrightness` 语义
- `sendRm(const RmCmd&)` — 按 `port` 选择 `Serial1` / `Serial2.write`
- `setVolume(uint8_t)` — 钳制 1..7 后 `vol_set`
- `refreshLamp()` — 亮度调节后刷新：`setBrightness(brightNess)` + 恢复 R/G/B + show

## Bug 修复（唯一行为变化）

原 `case 21/22` 亮度 ±25 用 `uint8_t` 直接运算，高/低亮度下会回绕：

- `brightNess = 250` 时 `250 + 25` → uint8 回绕成 19（反而变暗）
- `brightNess = 20` 时 `20 - 25` → 回绕成 251（反而变亮）

改为 `int` 中间变量正确钳制：亮一点 ≤ 255，暗一点 ≥ 10。

## 验证清单

- 表行保留原注释，可对照原代码 diff
- `static_assert` 校验表行数（16 / 60），防漏行
- 脚本核对每个 snid 恰好出现一次（颜色表 + 指令表 + 特殊逻辑）
- 用户在 IDE 中编译实测

## 覆盖的 snid 全集

1–25、28–36、37–80、161–178（26/27 为唤醒词，不在 switch 内），共 96 个，无重复、无遗漏。
