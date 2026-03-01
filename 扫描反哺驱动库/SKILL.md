---
name: 扫描反哺驱动库
description: 扫描当前项目的驱动文件，自动识别主控类型，将项目中有但 skill references 里没有的驱动反哺到对应库
context: main
---

# 扫描反哺驱动库

## 用户输入

$ARGUMENTS

> 如果用户传入了项目路径则使用该路径，否则使用当前工作目录。

---

## 执行流程

### 第一步：识别主控类型

扫描项目目录，按以下特征自动判断：

**STM32 HAL 项目特征（满足任意一条即判定）：**
- 存在 `Core/Src/` 或 `Core/Inc/` 目录
- 存在 `.ioc` 文件
- 存在 `main.c` 且内含 `HAL_Init()` 或 `MX_GPIO_Init()`
- 存在 `APP/` 目录且包含 `app_*.c` 文件
- 存在 `Drivers/STM32` 目录

**ESP32 Arduino 项目特征（满足任意一条即判定）：**
- 存在 `.ino` 文件
- 存在 `arduino.ino` 或 `*.ino`
- 存在 `app_*.cpp` 或 `drv_*.cpp` 文件
- 存在 `FangmuMqttLib.cpp` / `PubSubClient` 等 Arduino 库文件

判断结果：
- 明确识别 → 直接进入第二步
- 无法识别 → 询问用户："未能自动识别主控类型，请确认是 STM32 HAL 还是 ESP32 Arduino 项目？"

---

### 第二步：扫描项目驱动文件

根据主控类型，扫描项目中的**驱动层**文件：

> **收录范围说明：只收录驱动层，其余层一律跳过。**
>
> | 层级 | 判断标准 | 典型文件名 | 处理 |
> |------|---------|-----------|------|
> | 驱动层 | 直接操作硬件寄存器/外设 API，`drv_` 或 `app_` 前缀 | `drv_dht11.cpp`、`drv_oled.cpp`、`app_relay.c` | ✅ 收录 |
> | 功能层 | 封装业务逻辑，`func_`、`task_`、`logic_`、`svc_` 前缀 | `func_alarm.cpp`、`task_monitor.c` | ❌ 跳过 |
> | 显示页面层 | 基于驱动封装的多页面/UI 逻辑，`disp_`、`page_`、`ui_` 前缀 | `disp_page.cpp`、`ui_home.cpp` | ❌ 跳过 |
>
> **注意**：OLED 底层驱动（`drv_oled.cpp`）属于驱动层，应收录；基于它封装的页面切换逻辑（`disp_page.cpp`）属于显示页面层，跳过。

**STM32 项目** — 扫描以下位置的驱动文件：
- `APP/app_*.c` 和 `APP/app_*.h`
- 提取器件名：去掉 `app_` 前缀和 `.c/.h` 后缀，得到器件关键词（如 `dht11`、`ssd1306`、`key`）

**ESP32 项目** — 扫描以下位置的驱动文件：
- `*.ino` 同级目录下的 `drv_*.cpp` / `drv_*.h`（优先）以及 `app_*.cpp` / `app_*.h`
- 提取器件名：去掉 `drv_` / `app_` 前缀和 `.cpp/.h` 后缀，得到器件关键词
- 跳过前缀为 `func_`、`disp_`、`task_`、`page_`、`ui_`、`logic_` 的文件

---

### 第三步：对比 skill references

根据主控类型，与对应 skill 的 references 目录对比：

- **STM32** → 对比 `C:\Users\Administrator\.claude\commands\stm32-hal\references\`
  - 已有内容：子目录名（如 `dht11/`）和 `_hal.c` 文件名（如 `dht11_hal.c`）
- **ESP32** → 对比 `C:\Users\Administrator\.claude\commands\esp32-arduino\references\`
  - 已有内容：子目录名（如 `dht11/`）和 `_ino.cpp` 文件名（如 `dht11_ino.cpp`）

对比规则：用项目器件关键词与 references 文件名/目录名做**模糊匹配**（包含即算有）。

输出对比结果表：

| 器件 | 项目中的文件 | references 中是否已有 |
|------|------------|----------------------|
| dht11 | APP/app_dht11.c | ✅ 已有（dht11/） |
| mpu6050 | APP/app_mpu6050.c | ❌ 未收录 |
| relay | APP/app_relay.c | ❌ 未收录 |

---

### 第四步：反哺未收录的驱动

对每个「❌ 未收录」的器件，执行以下操作：

**STM32 项目 → stm32-hal references：**

1. 在 `C:\Users\Administrator\.claude\commands\stm32-hal\references\{器件名}\` 创建子目录
2. 将项目中该器件的 `.c` 和 `.h` 文件复制进去
3. 在 `C:\Users\Administrator\.claude\commands\stm32-hal\references\` 根目录追加一个 `{器件名}_hal.c` 摘要文件
   - 摘要内容：读取刚复制的 .c/.h 文件，提炼以下要点写入：
     - 总线类型
     - 关键 API 函数名和说明
     - 重要时序或注意事项（2-3条）
     - 依赖的 HAL 外设句柄

**ESP32 项目 → esp32-arduino references：**

1. 在 `C:\Users\Administrator\.claude\commands\esp32-arduino\references\{器件名}\` 创建子目录
2. 将项目中该器件的 `.cpp` 和 `.h` 文件复制进去
3. 在 `C:\Users\Administrator\.claude\commands\esp32-arduino\references\` 根目录追加一个 `{器件名}_ino.cpp` 摘要文件
   - 摘要内容：读取刚复制的 .cpp/.h 文件，提炼以下要点写入：
     - 总线类型
     - 关键 API 函数名和说明
     - 重要时序或注意事项（2-3条）
     - 库依赖（如有）

---

### 第五步：更新 references/README.md

将新收录的器件追加到对应 skill 的 `references/README.md` 的「已有完整例程」表格中。

格式：
```
| {器件名}/ | {器件描述} | {文件名}.c/.h 或 .cpp/.h | 来源：{项目名} |
```

---

### 第六步：输出汇总

输出操作结果：

```
扫描完成 — {项目名}（{主控类型}）

新增收录到 references：
  ✅ {器件1}  →  references/{器件1}/
  ✅ {器件2}  →  references/{器件2}/

已有，跳过：
  — {器件3}（已收录）
  — {器件4}（已收录）

共扫描 N 个驱动，新增 M 个到 skill 资料库。
```

---

## 注意事项

- 复制文件时不覆盖已有内容，若目录已存在则跳过并提示
- 摘要文件（`_hal.c` / `_ino.cpp`）只提炼关键信息，不是完整代码，控制在 60 行以内
- 如果项目驱动文件命名不规范（不带 `app_` / `drv_` 前缀），也尝试匹配，并在输出中注明文件名
- README.md 追加时检查是否已有该行，避免重复写入
