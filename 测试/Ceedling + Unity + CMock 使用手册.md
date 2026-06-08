# Ceedling + Unity + CMock 使用手册

适用场景：Windows 11 开发机、目标代码最终运行在 MCU/嵌入式平台、当前先做**主机端单元测试**。

核心思路：**不是把测试烧进控制器跑**，而是在 Windows 电脑上用 GCC 编译你的 C 模块，把硬件、CAN 驱动、RTE、EEPROM、定时器等外部依赖用 CMock 替换掉，只验证本模块逻辑。Ceedling 官方定位就是 C 项目的构建/单元测试系统，尤其适合嵌入式 C 项目；它把 Unity、CMock 等组件组合在一起使用。([GitHub](https://github.com/ThrowTheSwitch/Ceedling "GitHub - ThrowTheSwitch/Ceedling: Unit testing and build system for C projects · GitHub"))

---

## 一、 三个工具分别干什么

|工具|作用|你需要记住的结论|
|---|---|---|
|**Unity**|C 语言断言框架|写 `TEST_ASSERT_EQUAL_UINT8()` 这类判断|
|**CMock**|根据 `.h` 头文件自动生成 mock 函数|替代 CAN、DIO、ADC、RTE、NvM 等外部接口|
|**Ceedling**|构建系统 + 测试运行器|负责扫描测试文件、生成 mock、编译、运行、输出结果|

单元测试本质上就是一个无参数、无返回值的 C 函数，里面放若干断言；断言失败后，该测试用例失败。Ceedling 使用 Unity 负责断言和测试结果统计。([throwtheswitch.github.io](https://throwtheswitch.github.io/Ceedling/latest/testing-guide/test-cases/ "How Does a Test Case Work? - Ceedling"))

例如：

```c
void test_add_should_return_5(void)
{
    TEST_ASSERT_EQUAL_INT(5, add(2, 3));
}
```

---

## 二、 推荐测试策略：先做“主机端单元测试”

你们是嵌入式产品，不能一开始就尝试在真实控制器上跑 Ceedling。建议分三层：

|层级|运行位置|目标|
|---|---|---|
|单元测试|Windows 主机|验证 C 函数、状态机、报文解析、边界条件|
|集成测试|开发板/目标板/仿真环境|验证模块组合、RTE、驱动适配|
|系统测试|实车/台架/HIL|验证完整产品功能|

Ceedling 初期只承担第一层：**主机端单元测试**。这样开发每改一次代码，几秒内就能跑一遍测试，不需要烧录硬件。

适合优先测试的模块：

```text
适合：
- CAN/J1939 报文打包、解析、信号缩放
- BCM 灯控状态机
- DCM 电机控制逻辑中的状态判断
- VCU 模式切换逻辑
- 故障诊断 debounce / confirm / recover
- 超时判断
- 参数边界处理
- RTE 之上的业务逻辑

不适合第一批：
- 时钟初始化
- 寄存器直接配置
- Flash 烧写底层
- 中断向量
- 芯片厂 HAL 初始化
```

---

## 三、 Windows 11 环境安装

### 3.1 安装内容

你需要安装：

```text
1. Ruby 3.x
2. Ceedling gem
3. GCC / MinGW 编译器
4. Git
5. VS Code 或其他编辑器
```

Ceedling 的官方安装方式是通过 RubyGems，要求 Ruby 3，并且 `gem install ceedling` 会安装 Ceedling 及其依赖；Ceedling 还需要命令行 C 工具链，默认可直接配合 GCC 使用，在 Windows 上官方建议使用 MinGW。([throwtheswitch.github.io](https://throwtheswitch.github.io/Ceedling/latest/getting-started/installation/ "Installation - Ceedling"))

### 3.2 安装 Ruby

推荐安装 **RubyInstaller + DevKit 3.x x64**。RubyInstaller 是 Windows 上常用的 Ruby 安装器，并集成 MSYS2/MINGW 工具链环境。([rubyinstaller.org](https://rubyinstaller.org/ "RubyInstaller for Windows"))

安装完成后，打开 **PowerShell** 或 **Git Bash**，检查：

```powershell
ruby -v
gem -v
```

应能看到版本号。

也可以用 winget：

```powershell
winget search RubyInstallerTeam.RubyWithDevKit
winget install RubyInstallerTeam.RubyWithDevKit.3.4
```

如果 `3.4` 不存在，就安装搜索结果里最新的 3.x 版本。Ruby 官方也给出了 Windows 下通过 winget 安装 RubyInstaller / RubyWithDevKit 的方式。([ruby-lang.org](https://www.ruby-lang.org/en/documentation/installation/ "Installing Ruby | Ruby"))

### 3.3 安装 Ceedling

```powershell
gem install ceedling
ceedling version
```

如果能看到 Ceedling 版本，说明安装完成。

### 3.4 安装 / 检查 GCC

检查：

```powershell
gcc --version
```

如果提示找不到 `gcc`，需要安装 MinGW/MSYS2 GCC，或者使用 RubyInstaller DevKit 附带的 MSYS2 工具链。

最简单判断标准：

```powershell
ruby -v
gem -v
ceedling version
gcc --version
```

这四个命令都能运行，环境基本可用。

---

## 四、 创建第一个 Ceedling 工程

新建工作目录：

```powershell
mkdir C:\work
cd C:\work
ceedling new --gitsupport can_unit_demo
cd can_unit_demo
```

Ceedling 支持用 `ceedling new <name>` 创建工程，也可以把 Ceedling 配置加入已有工程；常用命令包括 `ceedling test:all`、`ceedling release`、`ceedling help` 等。([GitHub](https://github.com/ThrowTheSwitch/Ceedling "GitHub - ThrowTheSwitch/Ceedling: Unit testing and build system for C projects · GitHub"))

初始目录大致如下：

```text
can_unit_demo/
├─ project.yml
├─ src/
├─ test/
├─ vendor/
└─ build/        # 运行测试后自动生成，通常不提交 Git
```

先运行：

```powershell
ceedling test:all
```

如果工程为空，可能显示没有测试或全部通过。接下来手工加一个模块。

---

## 五、 第一个纯函数单元测试：BCM 灯控命令解析

### 5.1 新建源文件

创建 `src/bcm_lamp.h`：

```c
#ifndef BCM_LAMP_H
#define BCM_LAMP_H

#include <stdint.h>

typedef enum
{
    BCM_LAMP_OFF   = 0u,
    BCM_LAMP_ON    = 1u,
    BCM_LAMP_ERROR = 2u
} BcmLampCmd_t;

BcmLampCmd_t BcmLamp_DecodeCmd(uint8_t raw);

#endif
```

创建 `src/bcm_lamp.c`：

```c
#include "bcm_lamp.h"

BcmLampCmd_t BcmLamp_DecodeCmd(uint8_t raw)
{
    switch (raw & 0x03u)
    {
        case 0u:
            return BCM_LAMP_OFF;

        case 1u:
            return BCM_LAMP_ON;

        default:
            return BCM_LAMP_ERROR;
    }
}
```

### 5.2 新建测试文件

创建 `test/test_bcm_lamp.c`：

```c
#include "unity.h"
#include "bcm_lamp.h"

void setUp(void)
{
    /* 每个测试用例执行前都会调用 */
}

void tearDown(void)
{
    /* 每个测试用例执行后都会调用 */
}

void test_BcmLamp_DecodeCmd_raw0_should_return_off(void)
{
    TEST_ASSERT_EQUAL_UINT8(BCM_LAMP_OFF, BcmLamp_DecodeCmd(0u));
}

void test_BcmLamp_DecodeCmd_raw1_should_return_on(void)
{
    TEST_ASSERT_EQUAL_UINT8(BCM_LAMP_ON, BcmLamp_DecodeCmd(1u));
}

void test_BcmLamp_DecodeCmd_raw2_should_return_error(void)
{
    TEST_ASSERT_EQUAL_UINT8(BCM_LAMP_ERROR, BcmLamp_DecodeCmd(2u));
}

void test_BcmLamp_DecodeCmd_raw3_should_return_error(void)
{
    TEST_ASSERT_EQUAL_UINT8(BCM_LAMP_ERROR, BcmLamp_DecodeCmd(3u));
}

void test_BcmLamp_DecodeCmd_upper_bits_should_be_ignored(void)
{
    TEST_ASSERT_EQUAL_UINT8(BCM_LAMP_ON, BcmLamp_DecodeCmd(0xF1u));
}
```

### 5.3 运行测试

```powershell
ceedling test:all
```

你应该看到类似：

```text
--------------------
OVERALL TEST SUMMARY
--------------------
TESTED:  5
PASSED:  5
FAILED:  0
IGNORED: 0
```

Unity 提供大量断言类型，包括布尔、整数、十六进制、位、范围、字符串、内存、数组和浮点等；数组断言会指出第一个不匹配的元素位置，适合报文 payload 校验。([GitHub](https://raw.githubusercontent.com/ThrowTheSwitch/Unity/master/docs/UnityAssertionsReference.md "raw.githubusercontent.com"))

常用断言：

```c
TEST_ASSERT_TRUE(x);
TEST_ASSERT_FALSE(x);

TEST_ASSERT_EQUAL_INT(expected, actual);
TEST_ASSERT_EQUAL_UINT8(expected, actual);
TEST_ASSERT_EQUAL_UINT16(expected, actual);
TEST_ASSERT_EQUAL_UINT32(expected, actual);

TEST_ASSERT_EQUAL_HEX8(expected, actual);
TEST_ASSERT_EQUAL_HEX16(expected, actual);
TEST_ASSERT_EQUAL_HEX32(expected, actual);

TEST_ASSERT_EQUAL_UINT8_ARRAY(expected_array, actual_array, length);

TEST_ASSERT_NULL(ptr);
TEST_ASSERT_NOT_NULL(ptr);
```

---

## 六、 CMock 入门：隔离 CAN 发送接口

真实代码中，业务逻辑经常会调用 CAN 驱动：

```c
CanIf_Transmit(id, data, dlc);
```

单元测试时不希望真的发 CAN 报文，因此要把 `CanIf_Transmit()` 替换成 mock。

CMock 会读取头文件中的函数原型，自动生成同名 mock 函数。你可以指定“这个函数应该被调用几次、参数是什么、返回什么”。如果调用次数、顺序或参数不符合预期，测试失败。([GitHub](https://raw.githubusercontent.com/ThrowTheSwitch/CMock/master/docs/CMock_Summary.md "raw.githubusercontent.com"))

---

## 七、 示例：测试 BCM 状态报文发送

### 7.1 定义被依赖接口

创建 `src/can_if.h`：

```c
#ifndef CAN_IF_H
#define CAN_IF_H

#include <stdint.h>
#include <stdbool.h>

bool CanIf_Transmit(uint32_t can_id, const uint8_t *data, uint8_t dlc);

#endif
```

注意：这里**只写头文件**，不写 `can_if.c`。测试时由 CMock 自动生成 `mock_can_if.c`。

### 7.2 创建被测模块

创建 `src/bcm_lamp_tx.h`：

```c
#ifndef BCM_LAMP_TX_H
#define BCM_LAMP_TX_H

#include <stdint.h>
#include <stdbool.h>
#include "bcm_lamp.h"

bool BcmLamp_SendStatus(BcmLampCmd_t cmd);

#endif
```

创建 `src/bcm_lamp_tx.c`：

```c
#include "bcm_lamp_tx.h"
#include "can_if.h"

#define BCM_LAMP_STATUS_CAN_ID  (0x18FF1001u)
#define BCM_LAMP_STATUS_DLC     (8u)

bool BcmLamp_SendStatus(BcmLampCmd_t cmd)
{
    uint8_t data[8] = {0u};

    data[0] = (uint8_t)cmd;
    data[1] = 0xAAu;  /* 示例：协议固定字节 */
    data[2] = 0u;
    data[3] = 0u;
    data[4] = 0u;
    data[5] = 0u;
    data[6] = 0u;
    data[7] = 0u;

    return CanIf_Transmit(BCM_LAMP_STATUS_CAN_ID, data, BCM_LAMP_STATUS_DLC);
}
```

### 7.3 配置 CMock 插件

打开 `project.yml`，找到或增加以下配置：

```yaml
:project:
  :use_mocks: TRUE

:cmock:
  :when_no_prototypes: :warn
  :plugins:
    - :ignore
    - :ignore_arg
    - :callback
    - :array
  :treat_as:
    uint8_t: HEX8
    uint16_t: HEX16
    uint32_t: HEX32
    bool: UINT8
```

CMock 的插件包括 `:ignore`、`:ignore_arg`、`:array`、`:callback`、`:return_thru_ptr` 等；`:array` 适合检查指针/数组参数，`:ignore_arg` 适合忽略某个参数，`:callback` 适合自定义检查逻辑。([GitHub](https://raw.githubusercontent.com/ThrowTheSwitch/CMock/master/docs/CMock_Summary.md "raw.githubusercontent.com"))

### 7.4 写 mock 测试

创建 `test/test_bcm_lamp_tx.c`：

```c
#include "unity.h"
#include "bcm_lamp_tx.h"
#include "mock_can_if.h"

static bool CanIf_Transmit_Callback(uint32_t can_id,
                                    const uint8_t *data,
                                    uint8_t dlc,
                                    int cmock_num_calls)
{
    uint8_t expected_data[8] = {
        BCM_LAMP_ON,
        0xAAu,
        0u, 0u, 0u, 0u, 0u, 0u
    };

    (void)cmock_num_calls;

    TEST_ASSERT_EQUAL_HEX32(0x18FF1001u, can_id);
    TEST_ASSERT_EQUAL_UINT8(8u, dlc);
    TEST_ASSERT_NOT_NULL(data);
    TEST_ASSERT_EQUAL_UINT8_ARRAY(expected_data, data, 8u);

    return true;
}

void setUp(void)
{
}

void tearDown(void)
{
}

void test_BcmLamp_SendStatus_should_pack_payload_and_call_CanIf(void)
{
    CanIf_Transmit_Stub(CanIf_Transmit_Callback);

    TEST_ASSERT_TRUE(BcmLamp_SendStatus(BCM_LAMP_ON));
}
```

运行：

```powershell
ceedling test:test_bcm_lamp_tx
```

如果你的 CMock 版本生成的函数不是 `CanIf_Transmit_Stub()`，打开自动生成的 mock 头文件查看实际名称：

```text
build/test/mocks/mock_can_if.h
```

有些版本或配置下可能叫：

```c
CanIf_Transmit_StubWithCallback(...)
```

这种情况下把测试代码改成对应名称即可。CMock 文档说明 callback 插件会生成 `func_AddCallback` / `func_Stub`，旧名称 `func_StubWithCallback` 仍可能出现。([GitHub](https://raw.githubusercontent.com/ThrowTheSwitch/CMock/master/docs/CMock_Summary.md "raw.githubusercontent.com"))

---

## 八、 不用 callback 的写法：ExpectWithArray

如果你只想验证 payload 是否等于某个数组，可以使用 `:array` 插件生成的 `ExpectWithArray` 系列函数。

示例：

```c
void test_BcmLamp_SendStatus_should_send_expected_payload(void)
{
    uint8_t expected_data[8] = {
        BCM_LAMP_ON,
        0xAAu,
        0u, 0u, 0u, 0u, 0u, 0u
    };

    CanIf_Transmit_ExpectWithArrayAndReturn(
        0x18FF1001u,
        expected_data,
        8,
        8u,
        true
    );

    TEST_ASSERT_TRUE(BcmLamp_SendStatus(BCM_LAMP_ON));
}
```

但对于新手，我更建议先用 callback。原因是 callback 里可以写普通 Unity 断言，逻辑更直观。

---

## 九、 嵌入式代码如何改造，才能被单元测试

你们现有底层 C 代码可能会这样写：

```c
void App_Task(void)
{
    if (HAL_GPIO_ReadPin(PORT_A, PIN_1) == 1)
    {
        CAN_Send(0x123, data, 8);
    }
}
```

这种代码不好测，因为它直接依赖硬件。

建议改成：

```c
#include "dio_if.h"
#include "can_if.h"

void App_Task(void)
{
    if (DioIf_ReadInput(DIO_INPUT_DOOR) == 1u)
    {
        uint8_t data[8] = {0u};
        CanIf_Transmit(0x123u, data, 8u);
    }
}
```

然后：

```text
产品代码：
DioIf_ReadInput() -> 调用真实 HAL

单元测试：
DioIf_ReadInput() -> CMock 生成的 mock
```

这就是嵌入式单元测试的关键：**业务逻辑不要直接碰寄存器、HAL、CAN 控制器、RTE 外部接口；中间加一层可 mock 的接口头文件。**

---

## 十、 推荐的工程目录结构

对于你们公司后续项目，建议这样组织：

```text
Product_BCM/
├─ app/                         # 应用逻辑，重点单元测试
│  ├─ bcm_lamp.c
│  ├─ bcm_lamp.h
│  ├─ bcm_door.c
│  └─ bcm_door.h
│
├─ bsw/                         # 基础软件适配层
│  ├─ can_if.h
│  ├─ dio_if.h
│  ├─ nvm_if.h
│  └─ os_if.h
│
├─ drivers/                     # 芯片驱动/HAL，不作为第一批单元测试重点
│
├─ rte/                         # RTE 连接层
│
├─ test/
│  ├─ test_bcm_lamp.c
│  ├─ test_bcm_door.c
│  └─ support/
│     ├─ test_platform.h
│     └─ fake_time.c
│
├─ project.yml
└─ .gitlab-ci.yml
```

`project.yml` 可配置成：

```yaml
:project:
  :use_mocks: TRUE
  :use_test_preprocessor: TRUE
  :build_root: build

:paths:
  :test:
    - +:test/**
    - -:test/support
  :source:
    - app/**
    - test/support/**
  :include:
    - app/**
    - bsw/**
    - rte/**
    - test/support/**

:defines:
  :common:
    - UNIT_TEST
    - HOST_TEST
    - WIN32
```

初期不要把 `drivers/**` 全部加入 `:source`，否则可能引入大量 MCU 专用寄存器、编译器关键字、中断声明，导致主机端 GCC 编译失败。

---

## 十一、 处理 MCU 专用代码

### 11.1 编译器关键字

目标编译器可能有这些关键字：

```c
__interrupt
__near
__far
__root
__ramfunc
__weak
__attribute__((section(".xxx")))
```

Windows GCC 未必认识。测试时可以在 `test/support/test_platform.h` 里屏蔽：

```c
#ifndef TEST_PLATFORM_H
#define TEST_PLATFORM_H

#ifdef UNIT_TEST

#define __interrupt
#define __near
#define __far
#define __root
#define __ramfunc
#define __weak
#define __STATIC_INLINE static inline

#endif

#endif
```

然后在 `project.yml` 里强制 include：

```yaml
:tools:
  :test_compiler:
    :arguments:
      - -include
      - test/support/test_platform.h
```

更好的方式是把目标编译器相关宏集中放到 `compiler_abstraction.h`，产品和测试共用不同定义。

CMock 也支持通过 `:attributes`、`:strippables` 等配置处理编译器扩展语法。([GitHub](https://raw.githubusercontent.com/ThrowTheSwitch/CMock/master/docs/CMock_Summary.md "raw.githubusercontent.com"))

### 11.2 volatile 寄存器

不要直接测试这种函数：

```c
void Dio_SetHigh(void)
{
    PORTA->OUT |= (1u << 3);
}
```

先包一层：

```c
void DioIf_SetOutput(DioChannel_t ch, DioLevel_t level)
{
    Dio_SetHighLow_ByHal(ch, level);
}
```

业务模块只调用 `DioIf_SetOutput()`。单元测试 mock `DioIf_SetOutput()`，底层寄存器留给集成测试或目标板测试。

### 11.3 时间、定时器、超时

不要在单元测试里真的等待：

```c
DelayMs(100);
```

改成：

```c
if (OsIf_GetTickMs() - start_time > timeout_ms)
{
    state = TIMEOUT;
}
```

测试时 mock `OsIf_GetTickMs()` 返回不同时间点。

---

## 十二、 CAN/J1939 模块怎么测

CAN/J1939 很适合单元测试。重点不是“能不能发到总线”，而是：

```text
1. CAN ID 是否正确
2. 是否使用扩展帧
3. DLC 是否正确
4. payload 字节是否正确
5. bit/byte 顺序是否正确
6. 信号缩放是否正确
7. 边界值是否处理
8. 非法值是否拒绝
9. 超时、丢帧、错误帧后的状态是否正确
```

### 12.1 报文打包测试示例

假设有函数：

```c
void Bcm_PackLampStatus(uint8_t *data, BcmLampCmd_t cmd, uint8_t fault);
```

产品代码：

```c
#include "bcm_msg.h"

void Bcm_PackLampStatus(uint8_t *data, BcmLampCmd_t cmd, uint8_t fault)
{
    data[0] = (uint8_t)cmd;
    data[1] = fault;
    data[2] = 0u;
    data[3] = 0u;
    data[4] = 0u;
    data[5] = 0u;
    data[6] = 0u;
    data[7] = 0u;
}
```

测试：

```c
#include "unity.h"
#include "bcm_msg.h"

void setUp(void) {}
void tearDown(void) {}

void test_Bcm_PackLampStatus_should_pack_on_without_fault(void)
{
    uint8_t actual[8] = {0xFFu};
    uint8_t expected[8] = {
        BCM_LAMP_ON,
        0u,
        0u, 0u, 0u, 0u, 0u, 0u
    };

    Bcm_PackLampStatus(actual, BCM_LAMP_ON, 0u);

    TEST_ASSERT_EQUAL_UINT8_ARRAY(expected, actual, 8u);
}

void test_Bcm_PackLampStatus_should_pack_fault(void)
{
    uint8_t actual[8] = {0u};
    uint8_t expected[8] = {
        BCM_LAMP_ERROR,
        1u,
        0u, 0u, 0u, 0u, 0u, 0u
    };

    Bcm_PackLampStatus(actual, BCM_LAMP_ERROR, 1u);

    TEST_ASSERT_EQUAL_UINT8_ARRAY(expected, actual, 8u);
}
```

### 12.2 J1939 PGN 提取测试示例

先写函数：

```c
#include <stdint.h>
#include <stdbool.h>

uint32_t J1939_GetPgn(uint32_t can_id_29bit)
{
    uint32_t edp_dp = (can_id_29bit >> 24) & 0x03u;
    uint32_t pf     = (can_id_29bit >> 16) & 0xFFu;
    uint32_t ps     = (can_id_29bit >> 8)  & 0xFFu;

    if (pf < 240u)
    {
        /* PDU1: PS 是目标地址，不属于 PGN */
        return (edp_dp << 16) | (pf << 8);
    }
    else
    {
        /* PDU2: PS 是 Group Extension，属于 PGN */
        return (edp_dp << 16) | (pf << 8) | ps;
    }
}
```

测试：

```c
#include "unity.h"
#include "j1939_id.h"

void setUp(void) {}
void tearDown(void) {}

void test_J1939_GetPgn_PDU1_should_clear_ps(void)
{
    uint32_t id = 0x18EC3301u; /* PF=0xEC, PS=0x33 */
    TEST_ASSERT_EQUAL_HEX32(0x00EC00u, J1939_GetPgn(id));
}

void test_J1939_GetPgn_PDU2_should_keep_ps(void)
{
    uint32_t id = 0x18FEF101u; /* PF=0xFE, PS=0xF1 */
    TEST_ASSERT_EQUAL_HEX32(0x00FEF1u, J1939_GetPgn(id));
}
```

这类测试可以覆盖大量报文边界，特别适合你们当前“报文测试缺失”的问题。

---

## 十三、 覆盖率

初期先不要追求 100% 覆盖率。建议目标：

```text
第 1 阶段：关键纯逻辑模块 40%~50%
第 2 阶段：核心状态机、报文编解码 60%~70%
第 3 阶段：新增代码必须带单元测试
```

Ceedling 有内置插件支持覆盖率、测试报告、CI 集成等能力。([GitHub](https://github.com/ThrowTheSwitch/Ceedling "GitHub - ThrowTheSwitch/Ceedling: Unit testing and build system for C projects · GitHub"))

可在 `project.yml` 中启用：

```yaml
:plugins:
  :enabled:
    - stdout_pretty_tests_report
    - module_generator
    - gcov
```

运行：

```powershell
ceedling gcov:all
```

如果 Windows 下 `gcov` 不可用，需要检查：

```powershell
gcov --version
```

GCC、gcov、gcovr 的路径要一致，否则覆盖率报告可能生成失败。

---

## 十四、 GitLab CI 集成

你们已经有 GitLab CE，建议先做到：

```text
每次 push / merge request：
1. 编译单元测试
2. 运行所有单元测试
3. 保存测试日志
4. 失败则禁止合并
```

`.gitlab-ci.yml` 示例，适用于 Windows Shell Runner：

```yaml
stages:
  - unit_test

unit_test:
  stage: unit_test
  tags:
    - windows
  script:
    - ruby -v
    - gcc --version
    - ceedling version
    - ceedling clobber
    - ceedling test:all
  artifacts:
    when: always
    paths:
      - build/
    expire_in: 2 weeks
```

如果后续要在 GitLab MR 页面直接显示测试结果，需要输出 JUnit XML。GitLab 的单元测试报告要求 JUnit XML 格式，并通过 `artifacts:reports:junit` 上传；报告可在 Merge Request 和 pipeline 页面中显示失败详情。([GitLab Docs](https://docs.gitlab.com/ci/testing/unit_test_reports/ "Unit test reports | GitLab Docs"))

示意：

```yaml
unit_test:
  stage: unit_test
  script:
    - ceedling test:all
  artifacts:
    when: always
    paths:
      - build/
    reports:
      junit: build/artifacts/test/report.xml
```

注意：上面的 `report.xml` 路径要以你实际生成的 XML 文件为准。不要盲目照抄路径。

覆盖率如果生成 Cobertura XML，可通过 GitLab `coverage_report` 上传，GitLab 支持 Cobertura 或 JaCoCo 格式的覆盖率报告。([GitLab Docs](https://docs.gitlab.com/ci/yaml/artifacts_reports/ "CI/CD artifacts reports types | GitLab Docs"))

---

## 十五、 用 AI 辅助写单元测试

AI 可以提高效率，但不能直接信任。建议作为“测试用例助手”和“代码可测性重构助手”。

### 15.1 让 AI 生成测试用例

提示词模板：

```text
下面是一个嵌入式 C 模块。请使用 Ceedling + Unity 为它生成单元测试。
要求：
1. 测试正常输入、边界输入、非法输入。
2. 使用 TEST_ASSERT_EQUAL_UINT8 / TEST_ASSERT_EQUAL_HEX32 / TEST_ASSERT_EQUAL_UINT8_ARRAY。
3. 测试文件命名为 test_xxx.c。
4. 不要修改产品代码，除非指出无法测试的原因。
5. 如果存在外部依赖，请用 CMock mock 掉。

头文件：
[粘贴 .h]

源文件：
[粘贴 .c]

功能规范：
[粘贴需求]
```

### 15.2 让 AI 找边界条件

```text
请基于以下 CAN/J1939 报文定义，列出应该做单元测试的边界条件。
请按以下格式输出：
- 测试名称
- 输入
- 期望输出
- 目的

报文定义：
[粘贴 DBC/Excel/协议描述]
```

### 15.3 让 AI 帮你改造代码可测试性

```text
下面的嵌入式 C 代码直接调用 HAL/CAN/RTE，导致无法单元测试。
请重构为可用 Ceedling + CMock 测试的形式。
要求：
1. 保持原有功能不变。
2. 抽象外部依赖到 xxx_if.h。
3. 给出产品代码和测试代码。
4. 说明 mock 哪些函数。

代码：
[粘贴代码]
```

### 15.4 AI 生成后必须人工检查

重点检查：

```text
1. 断言是否真的验证了需求
2. 是否只测了“快乐路径”
3. 是否遗漏边界值
4. 是否把产品 bug 误当成正确行为
5. mock 的返回值是否符合真实驱动行为
6. 是否为了通过测试而修改了需求含义
```

---

## 十六、 团队落地建议

### 16.1 第一批不要全项目铺开

选 1 个模块试点。建议优先级：

```text
1. CAN/J1939 报文 pack/unpack
2. 故障状态机
3. 输入 debounce
4. 超时检测
5. 灯控/门控/继电器控制策略
```

不要从 MCAL、HAL、芯片初始化开始。

### 16.2 每个模块至少要求这些测试

```text
1. 正常值
2. 最小值
3. 最大值
4. 非法值
5. 空指针，如果函数允许传指针
6. 状态切换
7. 依赖函数失败
8. 超时
9. 报文 DLC 错误
10. 保留位/无效枚举值
```

### 16.3 代码提交规范

后续 GitLab MR 可以加规则：

```text
1. 新增 C 模块必须有 test_xxx.c
2. 修改报文解析逻辑必须更新对应测试
3. 修改状态机必须补状态切换测试
4. 所有 ceedling test:all 必须通过
5. 代码 review 时必须看测试是否覆盖异常路径
```

---

## 十七、 常见错误处理

|错误|可能原因|处理|
|---|---|---|
|`ruby 不是内部或外部命令`|Ruby 未加入 PATH|重新安装 RubyInstaller，勾选 PATH|
|`ceedling 不是内部或外部命令`|gem bin 路径未加入 PATH|关闭终端重开，或检查 Ruby gem 路径|
|`gcc 不是内部或外部命令`|未安装 MinGW/GCC|安装 MSYS2/MinGW，加入 PATH|
|`undefined reference to xxx`|被测模块调用了外部函数但没 mock|新建对应 `.h`，测试里 include `mock_xxx.h`|
|`Function xxx called more times than expected`|mock 期望次数不对|增加 Expect，或改用 Ignore/Callback|
|`Function xxx called fewer times than expected`|产品代码没有按预期调用依赖|检查逻辑分支或测试输入|
|`Expected 0x01 Was 0x00`|断言失败|先确认需求，再判断是代码 bug 还是测试写错|
|CMock 解析头文件失败|目标编译器关键字无法识别|用 `:strippables`、`:attributes` 或测试宏屏蔽|
|测试在 Windows 通过，目标板不通过|主机编译器与目标编译器差异|单元测试只保证逻辑，仍需集成/目标测试|

---

## 十八、 最小工作流

每天开发时：

```powershell
# 1. 写或修改代码
# 2. 写或修改测试
ceedling test:all

# 3. 看覆盖率，可选
ceedling gcov:all

# 4. 提交
git add .
git commit -m "add unit tests for bcm lamp status"
git push
```

建议开发人员形成习惯：

```text
不是“软件写完后交给测试测”，
而是“每写一个模块，就给它留下可重复运行的单元测试”。
```

你们公司当前最值得先做的不是复杂 HIL，也不是一开始买昂贵工具，而是先把 **Ceedling + Unity + CMock + GitLab CI** 跑起来。第一批目标只要做到：报文编解码、状态机、异常路径能自动回归，就能明显减少系统测试压力。