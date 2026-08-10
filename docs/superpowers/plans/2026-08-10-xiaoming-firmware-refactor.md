# xiaoMing 固件重构实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 `Firmware/xiaoming.hd` 里约 90 个 case 的巨型 `switch` 重构为「数据驱动查表 + 特殊逻辑 handler」的三步分发，同时修复亮度调节的 uint8 回绕 bug。

**Architecture:** 两张静态表（颜色表 16 行、红外/433 指令表 60 行）+ 4 个辅助函数 + 一个特殊逻辑 switch。`ASR_CODE()` 变为：查颜色表 → 查指令表 → 特殊逻辑。行为 100% 保持（唯一例外：亮度钳制 bug 修复）。

**Tech Stack:** ASRPRO pro-code 单文件 C++（天问 Block IDE 编译），仅改动 `Firmware/xiaoming.hd`。

## Global Constraints

- 保持 ASRPRO pro-code 单文件格式：保留 includes、全局变量、`setup()` 中 `{ID:xx,...}` 语音定义注释、`/** edittype="asr_procode" */` 尾部标记、标准入口（`setup` / `ASR_CODE` / `hardware_init`）。
- `hardware_init()` 与 `setup()` 不得改动（字节级保持原样）。
- 本地无 ASRPRO 编译环境，无法在本地编译；验证靠静态脚本 + 用户到 IDE 编译实测。
- 行为 100% 保持，唯一允许的行为变化是亮度钳制 bug 修复（见 Task 2）。
- Commit 格式：`@操作(关键词): 描述` + 末尾空行 `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`。

---

### Task 1: 新增数据结构和两张静态表

**Files:**
- Modify: `Firmware/xiaoming.hd`（在 `WS2812 ASR_WS2812(4);` 之后、`timerSleepProc` 函数之前插入）

**Interfaces:**
- Produces:
  - `struct ColorCmd { uint32_t snid; uint8_t r, g, b; uint8_t brightness; bool fixedBrightness; bool applyBrightness; }`
  - `struct RmCmd { uint32_t snid; uint8_t port; const char* cmd; }`
  - `static const ColorCmd COLOR_TABLE[]`（16 行）
  - `static const RmCmd RM_TABLE[]`（60 行）
  - 宏 `ARRAY_SIZE(a)`、`STATIC_ASSERT(cond, name)`

- [ ] **Step 1: 插入结构与表**

在 `WS2812 ASR_WS2812(4);` 后、`void timerSleepProc(...)` 前插入以下内容（原样粘贴）：

```cpp
// ============================================================
// 数据驱动：颜色指令表
//   fixedBrightness=true  → 用 brightness 固定亮度
//   fixedBrightness=false → 用当前 brightNess
//   applyBrightness=false → 不调用 setBrightness（如关灯）
// ============================================================
struct ColorCmd {
  uint32_t snid;
  uint8_t  r, g, b;
  uint8_t  brightness;      // fixedBrightness 为 true 时生效
  bool     fixedBrightness;
  bool     applyBrightness;
};

static const ColorCmd COLOR_TABLE[] = {
  // ---- 灯光 ----
  {  1, 248, 141,  30,   0, false, true },  // 开灯
  {  2, 255,   0,   0,   0, false, true },  // 红灯
  {  3,   0,   0, 255,   0, false, true },  // 蓝灯
  {  4,   0, 255,   0,   0, false, true },  // 绿灯
  {  5, 248, 215,  20,   0, false, true },  // 黄灯
  {  6, 255, 255, 255,   0, false, true },  // 白灯
  {  7, 248, 141,  30,   0, false, true },  // 暖光灯
  {  8, 248, 141,  30,  10,  true, true },  // 夜灯（固定亮度10）
  {  9,   0,   0,   0,   0, false, false }, // 关灯（不设亮度）
  // ---- 场景 ----
  { 10, 227, 171,  87, 225,  true, true },  // 日出模式
  { 11, 248, 141,  30, 255,  true, true },  // 下午茶模式
  { 12, 255, 255, 255, 200,  true, true },  // 阅读模式
  // ---- 更多颜色 ----
  { 33,  73,   6,  75,   0, false, true },  // 紫灯
  { 34, 243, 139,   0,   0, false, true },  // 橙灯
  { 35,  17, 211, 188,   0, false, true },  // 青灯
  { 36, 251, 146, 158,   0, false, true },  // 粉灯
};

// ============================================================
// 数据驱动：红外 / 433 学习与发射指令表
//   port=1 → Serial1（红外），port=2 → Serial2（433）
//   cmd 为 4 字节 ASCII："xx"前缀=学习，"fs"前缀=发射
// ============================================================
struct RmCmd {
  uint32_t snid;
  uint8_t  port;
  const char* cmd;
};

static const RmCmd RM_TABLE[] = {
  // ---- 红外 · 空调（Serial1）----
  {  37, 1, "xx00" },  // 学习打开空调
  {  38, 1, "xx02" },  // 学习关闭空调
  {  39, 1, "fs00" },  // 打开空调
  {  40, 1, "fs02" },  // 关闭空调
  {  41, 1, "xx04" },  // 学习空调二十五度
  {  42, 1, "xx06" },  // 学习空调二十四度
  {  43, 1, "xx08" },  // 学习空调二十三度
  {  44, 1, "xx10" },  // 学习空调二十二度
  {  45, 1, "xx12" },  // 学习空调二十一度
  {  46, 1, "fs04" },  // 空调二十五度
  {  47, 1, "fs06" },  // 空调二十四度
  {  48, 1, "fs08" },  // 空调二十三度
  {  49, 1, "fs10" },  // 空调二十二度
  {  50, 1, "fs12" },  // 空调二十一度
  // ---- 红外 · 氛围灯（Serial1）----
  {  51, 1, "xx14" },  // 学习打开氛围灯
  {  52, 1, "xx16" },  // 学习关闭氛围灯
  {  53, 1, "fs14" },  // 打开氛围灯
  {  54, 1, "fs16" },  // 关闭氛围灯
  {  55, 1, "xx18" },  // 学习氛围灯变亮
  {  56, 1, "xx20" },  // 学习氛围灯变暗
  {  57, 1, "fs18" },  // 氛围灯变亮
  {  58, 1, "fs20" },  // 氛围灯变暗
  {  59, 1, "xx22" },  // 学习氛围灯红色
  {  60, 1, "xx24" },  // 学习氛围灯绿色
  {  61, 1, "xx26" },  // 学习氛围灯蓝色
  {  62, 1, "xx28" },  // 学习氛围灯橙色
  {  63, 1, "xx30" },  // 学习氛围灯青色
  {  64, 1, "xx32" },  // 学习氛围灯紫色
  {  65, 1, "xx34" },  // 学习氛围灯粉色
  {  66, 1, "xx36" },  // 学习氛围灯黄色
  {  67, 1, "xx38" },  // 学习氛围灯白色
  {  68, 1, "fs22" },  // 氛围灯红色
  {  69, 1, "fs24" },  // 氛围灯绿色
  {  70, 1, "fs26" },  // 氛围灯蓝色
  {  71, 1, "fs28" },  // 氛围灯橙色
  {  72, 1, "fs30" },  // 氛围灯青色
  {  73, 1, "fs32" },  // 氛围灯紫色
  {  74, 1, "fs34" },  // 氛围灯粉色
  {  75, 1, "fs36" },  // 氛围灯黄色
  {  76, 1, "fs38" },  // 氛围灯白色
  {  77, 1, "xx40" },  // 学习氛围灯流水
  {  78, 1, "xx42" },  // 学习流水氛围灯
  {  79, 1, "fs40" },  // 氛围灯流水
  {  80, 1, "fs42" },  // 流水氛围灯
  // ---- 433 · 桌灯 / 大灯（Serial2）----
  { 161, 2, "xx00" },  // 学习打开桌灯
  { 162, 2, "xx02" },  // 学习关闭桌灯
  { 163, 2, "fs00" },  // 打开桌灯
  { 164, 2, "fs02" },  // 关闭桌灯
  { 165, 2, "xx04" },  // 学习桌灯亮一点
  { 166, 2, "xx06" },  // 学习桌灯变暖
  { 167, 2, "fs04" },  // 桌灯亮一点
  { 168, 2, "fs06" },  // 桌灯变暖
  { 169, 2, "xx08" },  // 学习桌灯暗一点
  { 170, 2, "xx10" },  // 学习桌灯变冷
  { 171, 2, "fs08" },  // 桌灯暗一点
  { 172, 2, "fs10" },  // 桌灯变冷
  { 173, 2, "xx12" },  // 学习打开大灯
  { 174, 2, "xx14" },  // 学习关闭大灯
  { 175, 2, "fs12" },  // 打开大灯
  { 176, 2, "fs14" },  // 关闭大灯
};

#define ARRAY_SIZE(a) (sizeof(a) / sizeof((a)[0]))
#define STATIC_ASSERT(cond, name) typedef char static_assert_##name[(cond) ? 1 : -1]
STATIC_ASSERT(ARRAY_SIZE(COLOR_TABLE) == 16, color_table_size);
STATIC_ASSERT(ARRAY_SIZE(RM_TABLE) == 60, rm_table_size);
```

- [ ] **Step 2: 验证表行数与无重复 snid**

运行（Git Bash）：

```bash
cd /c/Users/11846/Desktop/Git_Code/TianwenCode/xiaoMing
python - <<'PY'
import re
src = open('Firmware/xiaoming.hd', encoding='utf-8').read()
ids = [int(m) for m in re.findall(r'^\s*\{\s*(\d+),', src, re.M)]
print('表行数 =', len(ids))
assert len(ids) == 76, '应共 76 行（颜色 16 + 指令 60）'
dup = sorted({x for x in ids if ids.count(x) > 1})
print('重复 snid =', dup)
assert not dup, '表内存在重复 snid'
print('OK: 行数与唯一性校验通过')
PY
```

期望输出：`表行数 = 76`、`重复 snid = []`、`OK: ...`。

- [ ] **Step 3: Commit**

```bash
git add Firmware/xiaoming.hd
git commit -m "@add(fw): 新增颜色/红外433指令数据表与结构体" -m "Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: 新增辅助函数 + 重写 ASR_CODE + 亮度 bug 修复

**Files:**
- Modify: `Firmware/xiaoming.hd`（把原 `void ASR_CODE(){...}` 整个巨型 switch 替换为下方的新版本）

**Interfaces:**
- Consumes: Task 1 的 `ColorCmd` / `RmCmd` / `COLOR_TABLE` / `RM_TABLE`
- Produces: `ASR_CODE()`（无返回值）、辅助函数 `lampSetColor` / `findAndApplyColor` / `findAndSendRm` / `setVolume` / `refreshLamp` / `handleSpecials`

- [ ] **Step 1: 替换 ASR_CODE 及辅助函数**

把原 `void ASR_CODE(){ ... }`（从 `void ASR_CODE(){` 到函数结束的 `}`，含全部 case）整段替换为：

```cpp
// ============================================================
// 辅助函数
// ============================================================

// 按固定亮度或当前亮度点亮灯带（applyBrightness=false 时跳过 setBrightness 与 R/G/B 记录）
// 注意：必须同步 R/G/B 全局（原代码每个颜色 case 都这么做），供"亮一点/暗一点/吃掉彩虹"重绘
static void lampSetColor(uint8_t r, uint8_t g, uint8_t b, uint8_t brightness, bool fixedBrightness, bool applyBrightness) {
  if (applyBrightness) {
    R = r; G = g; B = b;
    uint8_t br = fixedBrightness ? brightness : brightNess;
    ASR_WS2812.setBrightness(br);
  }
  ASR_WS2812.pixel_set_all_color(r, g, b);
  ASR_WS2812.pixel_show();
}

// 颜色表查询
static bool findAndApplyColor(uint32_t id) {
  for (uint32_t i = 0; i < ARRAY_SIZE(COLOR_TABLE); i++) {
    if (COLOR_TABLE[i].snid == id) {
      const ColorCmd &c = COLOR_TABLE[i];
      lampSetColor(c.r, c.g, c.b, c.brightness, c.fixedBrightness, c.applyBrightness);
      return true;
    }
  }
  return false;
}

// 红外 / 433 指令表查询
static bool findAndSendRm(uint32_t id) {
  for (uint32_t i = 0; i < ARRAY_SIZE(RM_TABLE); i++) {
    if (RM_TABLE[i].snid == id) {
      const RmCmd &c = RM_TABLE[i];
      if (c.port == 1)      Serial1.write(c.cmd);
      else if (c.port == 2) Serial2.write(c.cmd);
      return true;
    }
  }
  return false;
}

// 音量设置（钳制 1..7）
static void setVolume(uint8_t v) {
  volume = (v > 7) ? 7 : ((v < 1) ? 1 : v);
  vol_set(volume);
}

// 亮度调节后刷新为当前 R/G/B
static void refreshLamp() {
  ASR_WS2812.setBrightness(brightNess);
  ASR_WS2812.pixel_set_all_color(R, G, B);
  ASR_WS2812.pixel_show();
}

// ============================================================
// 特殊逻辑：音量 / 亮度 / 彩虹 / 睡眠定时 / 回家出门
// ============================================================
static void handleSpecials(uint32_t id) {
  switch (id) {
    // 音量
    case 14: case 15: setVolume(volume + 1); break;   // 大声点 / 声音大一点
    case 16: case 29: setVolume(7);          break;   // 最大声 / 最高音量
    case 17: case 18: setVolume(volume - 1); break;   // 小声点 / 声音小一点
    case 19: case 30: setVolume(1);          break;   // 最小声 / 最低音量
    case 20: setVolume(4);                   break;   // 正常音量
    // 亮度（用 int 中间值钳制，修复 uint8 回绕 bug）
    case 21: brightNess = ((int)brightNess + 25 > 255) ? 255 : brightNess + 25; refreshLamp(); break;  // 亮一点
    case 22: brightNess = ((int)brightNess - 25 <  10) ?  10 : brightNess - 25; refreshLamp(); break;  // 暗一点
    case 23: brightNess = 128; refreshLamp(); break;   // 正常亮度
    case 24: brightNess = 255; refreshLamp(); break;   // 最高亮度
    case 25: brightNess =  10; refreshLamp(); break;   // 最低亮度
    // 彩虹
    case 31: vTaskResume(rainbowProcHandler); break;   // 彩虹
    case 32: vTaskSuspend(rainbowProcHandler); lampSetColor(R, G, B, brightNess, false, true); break;  // 吃掉彩虹
    // 睡眠定时
    case 13: lampSetColor(248, 141, 30, 10, true, true); xTimerStart(timerSleepHandler, 0); break;  // 睡眠模式
    case 28: xTimerStop(timerSleepHandler, 0); break;  // 解除定时
    // 回家 / 出门（待实现）
    case 177: /* 回家模式 */ break;
    case 178: /* TODO：关闭空调，关闭大灯，关闭氛围灯，打开暖光灯 */ break;
    default: break;
  }
}

/* 语音识别成功钩子程序：查颜色表 → 查指令表 → 特殊逻辑 */
void ASR_CODE() {
  // 唤醒时间设置必须在 ASR_CODE 中才有效
  set_state_enter_wakeup(20000);

  if (findAndApplyColor(snid)) return;   // 1) 多色灯光 / 场景
  if (findAndSendRm(snid))      return;   // 2) 红外 / 433 学习与发射
  handleSpecials(snid);                  // 3) 音量 / 亮度 / 彩虹 / 定时等
}
```

注意：被替换掉的巨型 switch 中，`hardware_init()` 与 `setup()` 保持不变。

- [ ] **Step 2: 确认旧巨型 switch 已移除**

运行：

```bash
grep -n "Serial1.write\|Serial2.write" Firmware/xiaoming.hd
```

期望：仅在 `findAndSendRm` 内部出现 2 处（`Serial1.write(c.cmd)` / `Serial2.write(c.cmd)`），其余旧 case 的逐行 `Serial1.write("xx..")` 全部消失。若还有残留，手动删除旧 switch。

- [ ] **Step 3: Commit**

```bash
git add Firmware/xiaoming.hd
git commit -m "@update(fw): 重构ASR_CODE为数据驱动查表分发并修复亮度回绕" -m "Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: 一致性验证与收尾

**Files:**
- Modify: 无（验证为主）；若脚本发现问题，修复 `Firmware/xiaoming.hd`

- [ ] **Step 1: 核对每个 snid 恰好被覆盖一次**

运行：

```bash
cd /c/Users/11846/Desktop/Git_Code/TianwenCode/xiaoMing
python - <<'PY'
import re
src = open('Firmware/xiaoming.hd', encoding='utf-8').read()

# 表内 snid（初始化行第一个数字，排除结构体定义等）
table_ids = set(int(m) for m in re.findall(r'^\s*\{\s*(\d+)\s*,\s*[0-9"]', src, re.M))
# 特殊逻辑 switch 的 case
spec_ids = set(int(m) for m in re.findall(r'\bcase\s+(\d+)\s*:', src))

# 期望全集 = 原 switch 的 96 个 snid（26/27 为唤醒词，不在 switch 内）
expected = set(range(1, 26)) | set(range(28, 37)) | set(range(37, 81)) | set(range(161, 179))

actual = table_ids | spec_ids
print('期望数量:', len(expected), '实际数量:', len(actual))
print('缺失:', sorted(expected - actual))
print('多余:', sorted(actual - expected))
print('重复:', sorted({x for x in actual if (x in table_ids and x in spec_ids)}))
assert expected == actual, 'snid 覆盖不一致，请检查！'
print('OK: 96 个 snid 全部覆盖且无重复')
PY
```

期望输出：`期望数量: 96 实际数量: 96`、三个列表均为空、`OK: ...`。

- [ ] **Step 2: 确认 hardware_init() 与 setup() 未被改动**

运行：

```bash
git diff --stat
git diff Firmware/xiaoming.hd | grep -E "^[-+].*(hardware_init|setPinFun|xTimerCreate|vTaskCreate|ID:)" || echo "hardware_init/setup 相关行无改动"
```

期望：`git diff` 中不应有对 `hardware_init()`、`setup()`、`{ID:...}` 语音定义行的改动（`grep` 无输出即通过）。

- [ ] **Step 3: 收尾提交（如第 1/2 步发现问题并修复）**

若无需修复，此步跳过；若修复了，按格式提交：

```bash
git add Firmware/xiaoming.hd
git commit -m "@fix(fw): 修正重构后的一致性检查问题" -m "Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

- [ ] **Step 4: 交付说明**

告知用户：重构完成，请用天问 Block IDE 打开 `Firmware/xiaoming.hd` 编译烧录实测。重点验证：颜色/场景、亮度（尤其高亮度下「亮一点」不再变暗）、音量、彩虹、睡眠定时、红外/433 学习与发射。
