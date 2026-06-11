# 嵌入式 C 接口抽象与硬件解耦编码指导手册

## 一、目标

本文档用于指导嵌入式 C 项目中如何通过 **“数据结构 + 函数指针表”** 的方式实现：

- 硬件抽象
    
- 驱动解耦
    
- 接口稳定
    
- 多硬件适配
    
- 单元测试友好
    
- 业务逻辑与底层硬件隔离

核心思想是：

> 上层模块只依赖抽象接口，不直接依赖具体硬件、寄存器、芯片型号或外设实现。

## 二、基本设计模式

### 2.1 数据结构 + 函数指针表

在 C 语言中，可以通过结构体和函数指针模拟面向对象中的接口和多态。

推荐形式：

```c
typedef struct uart uart_t;

typedef struct {
    int (*init)(uart_t *uart);
    int (*deinit)(uart_t *uart);
    int (*send)(uart_t *uart, const uint8_t *data, uint32_t len);
    int (*recv)(uart_t *uart, uint8_t *buf, uint32_t len);
} uart_ops_t;

struct uart {
    const uart_ops_t *ops;
    void *priv;
};
```

调用方式：

```c
uart->ops->send(uart, data, len);
```

其中 `uart` 类似 C++ 的 `this` 或 Python 成员函数中的 `self`。

## 三、分层原则

推荐分层：

```text
Application / Business Logic
        |
        v
Abstract Interface Layer
        |
        v
Driver / HAL Adapter Layer
        |
        v
BSP / Register / Vendor SDK
        |
        v
Hardware
```

### 3.1 上层只依赖抽象接口

业务层不应直接访问：

```c
USART1->DR = data;
GPIOA->ODR |= GPIO_PIN_5;
HAL_UART_Transmit(...);
```

业务层应调用稳定抽象接口：

```c
uart_send(console_uart, data, len);
gpio_set(led_gpio, GPIO_LEVEL_HIGH);
```

## 四、接口定义规范

### 4.1 抽象接口头文件

例如 `uart_if.h`：

```c
#ifndef UART_IF_H
#define UART_IF_H

#include <stdint.h>

typedef struct uart uart_t;

typedef struct {
    int (*init)(uart_t *uart);
    int (*deinit)(uart_t *uart);
    int (*send)(uart_t *uart, const uint8_t *data, uint32_t len);
    int (*recv)(uart_t *uart, uint8_t *buf, uint32_t len);
} uart_ops_t;

struct uart {
    const uart_ops_t *ops;
    void *priv;
};

int uart_init(uart_t *uart);
int uart_deinit(uart_t *uart);
int uart_send(uart_t *uart, const uint8_t *data, uint32_t len);
int uart_recv(uart_t *uart, uint8_t *buf, uint32_t len);

#endif
```

### 4.2 接口包装函数

不要让上层到处直接调用 `ops`。

推荐提供统一包装函数：

```c
int uart_send(uart_t *uart, const uint8_t *data, uint32_t len)
{
    if (uart == NULL || uart->ops == NULL || uart->ops->send == NULL) {
        return -1;
    }

    return uart->ops->send(uart, data, len);
}
```

这样可以统一处理：

- 空指针检查
    
- 参数校验
    
- 日志
    
- 错误码转换
    
- 统计信息
    
- 调试钩子

## 五、硬件相关内容放到私有层

### 5.1 私有上下文结构

硬件相关内容不应暴露给上层。

例如 STM32 UART 实现：

```c
typedef struct {
    USART_TypeDef *instance;
    uint32_t baudrate;
    uint32_t irq_num;
} stm32_uart_priv_t;
```

然后挂到抽象对象中：

```c
static stm32_uart_priv_t uart1_priv = {
    .instance = USART1,
    .baudrate = 115200,
    .irq_num = USART1_IRQn,
};

uart_t uart1 = {
    .ops = &stm32_uart_ops,
    .priv = &uart1_priv,
};
```

上层只看到：

```c
extern uart_t uart1;
uart_send(&uart1, data, len);
```

## 六、使用 ops / vtable 表达不同硬件实现

### 6.1 STM32 实现

```c
static int stm32_uart_send(uart_t *uart, const uint8_t *data, uint32_t len)
{
    stm32_uart_priv_t *priv = (stm32_uart_priv_t *)uart->priv;

    for (uint32_t i = 0; i < len; i++) {
        while (!(priv->instance->SR & USART_SR_TXE)) {
        }

        priv->instance->DR = data[i];
    }

    return 0;
}

static const uart_ops_t stm32_uart_ops = {
    .init = stm32_uart_init,
    .deinit = stm32_uart_deinit,
    .send = stm32_uart_send,
    .recv = stm32_uart_recv,
};
```

### 6.2 模拟 UART 实现

```c
typedef struct {
    uint8_t *tx_buffer;
    uint32_t tx_size;
} mock_uart_priv_t;

static int mock_uart_send(uart_t *uart, const uint8_t *data, uint32_t len)
{
    mock_uart_priv_t *priv = (mock_uart_priv_t *)uart->priv;

    if (len > priv->tx_size) {
        return -1;
    }

    memcpy(priv->tx_buffer, data, len);
    return 0;
}

static const uart_ops_t mock_uart_ops = {
    .init = mock_uart_init,
    .deinit = mock_uart_deinit,
    .send = mock_uart_send,
    .recv = mock_uart_recv,
};
```

相同业务代码可以使用不同实现：

```c
uart_send(&uart1, data, len);      // 真实硬件
uart_send(&mock_uart, data, len);  // 单元测试
```

## 七、`void *priv` 与具体私有结构

### 7.1 推荐用法

抽象层使用：

```c
void *priv;
```

具体驱动层转换：

```c
stm32_uart_priv_t *priv = (stm32_uart_priv_t *)uart->priv;
```

优点：

- 抽象层不依赖具体硬件类型
    
- 不同芯片可以使用不同私有数据
    
- 同一个接口可适配多个硬件实现

### 7.2 更强类型安全的替代方案

如果项目规模较小，也可以使用具体类型：

```c
typedef struct {
    const uart_ops_t *ops;
    stm32_uart_priv_t *priv;
} stm32_uart_t;
```

但这种方式会降低跨平台抽象能力。

通用驱动框架更推荐使用 `void *priv`。

## 八、接口稳定，硬件实现可替换

接口一旦发布，应尽量保持稳定。

推荐：

```c
int uart_send(uart_t *uart, const uint8_t *data, uint32_t len);
```

不推荐：

```c
int stm32_uart_send(USART_TypeDef *uart, uint8_t *data, uint32_t len);
```

前者面向抽象对象，后者直接绑定 STM32 寄存器类型。

稳定接口的好处：

- 换芯片时业务层不变
    
- 换 HAL 库时业务层不变
    
- 单元测试可以替换 mock 实现
    
- 多个硬件平台可以共用业务代码

## 九、避免硬件细节污染业务逻辑

### 9.1 不推荐

```c
void app_led_on(void)
{
    GPIOA->ODR |= GPIO_PIN_5;
}
```

问题：

- 业务代码绑定具体 GPIO 端口
    
- 难以迁移芯片
    
- 难以单元测试
    
- 硬件细节扩散

### 9.2 推荐

```c
void app_led_on(gpio_t *led)
{
    gpio_set(led, GPIO_LEVEL_HIGH);
}
```

底层实现：

```c
static int stm32_gpio_set(gpio_t *gpio, gpio_level_t level)
{
    stm32_gpio_priv_t *priv = (stm32_gpio_priv_t *)gpio->priv;

    if (level == GPIO_LEVEL_HIGH) {
        priv->port->BSRR = priv->pin;
    } else {
        priv->port->BSRR = (uint32_t)priv->pin << 16;
    }

    return 0;
}
```

## 十、宏的使用规范

### 10.1 不推荐滥用宏

```c
#define LED_ON()  GPIOA->ODR |= GPIO_PIN_5
#define LED_OFF() GPIOA->ODR &= ~GPIO_PIN_5
```

问题：

- 没有类型检查
    
- 难以调试
    
- 容易隐藏副作用
    
- 破坏模块边界

### 10.2 推荐使用函数或接口

```c
int gpio_set(gpio_t *gpio, gpio_level_t level);
```

宏可以用于：

- 编译期配置
    
- 常量定义
    
- 条件编译
    
- 寄存器位定义

不建议用宏承载复杂业务逻辑或硬件操作流程。

## 十一、全局变量使用规范

### 11.1 不推荐

```c
extern UART_HandleTypeDef huart1;

void app_send_log(const char *msg)
{
    HAL_UART_Transmit(&huart1, msg, strlen(msg), 100);
}
```

问题：

- 业务层依赖全局硬件对象
    
- 模块耦合严重
    
- 测试困难
    
- 多实例扩展困难

### 11.2 推荐依赖注入

```c
typedef struct {
    uart_t *log_uart;
} app_context_t;

void app_send_log(app_context_t *ctx, const char *msg)
{
    uart_send(ctx->log_uart, (const uint8_t *)msg, strlen(msg));
}
```

这样业务层只依赖抽象接口。

## 十二、错误码规范

推荐统一错误码：

```c
typedef enum {
    DRIVER_OK = 0,
    DRIVER_ERR = -1,
    DRIVER_ERR_INVALID_ARG = -2,
    DRIVER_ERR_TIMEOUT = -3,
    DRIVER_ERR_BUSY = -4,
    DRIVER_ERR_NOT_SUPPORTED = -5,
} driver_err_t;
```

驱动接口统一返回 `int` 或 `driver_err_t`。

不推荐不同驱动返回不同风格：

```c
HAL_StatusTypeDef
bool
uint8_t
int
```

应在适配层统一转换。

## 十三、初始化与生命周期管理

### 13.1 推荐模式

```c
int uart_init(uart_t *uart)
{
    if (uart == NULL || uart->ops == NULL || uart->ops->init == NULL) {
        return DRIVER_ERR_INVALID_ARG;
    }

    return uart->ops->init(uart);
}
```

对象生命周期建议明确：

```text
定义对象 -> 绑定 ops -> 绑定 priv -> init -> 使用 -> deinit
```

示例：

```c
uart_t uart1 = {
    .ops = &stm32_uart_ops,
    .priv = &uart1_priv,
};

uart_init(&uart1);
uart_send(&uart1, data, len);
uart_deinit(&uart1);
```

## 十四、多态行为示例

同一套上层代码：

```c
void log_write(uart_t *uart, const char *msg)
{
    uart_send(uart, (const uint8_t *)msg, strlen(msg));
}
```

可以运行在不同底层实现上：

```c
log_write(&stm32_uart1, "hello");
log_write(&esp32_uart0, "hello");
log_write(&mock_uart, "hello");
```

这就是 C 语言中通过函数指针表实现的运行时多态。

## 十五、文件组织建议

推荐目录结构：

```text
project/
├── app/
│   └── app_log.c
├── interface/
│   ├── uart_if.h
│   └── gpio_if.h
├── driver/
│   ├── uart/
│   │   ├── stm32_uart.c
│   │   └── mock_uart.c
│   └── gpio/
│       ├── stm32_gpio.c
│       └── mock_gpio.c
├── bsp/
│   ├── board.c
│   └── board.h
└── main.c
```

职责划分：

|层级|职责|
|---|---|
|app|业务逻辑|
|interface|抽象接口|
|driver|具体驱动实现|
|bsp|板级资源绑定|
|main|系统启动与装配|

## 十六、BSP 负责对象装配

BSP 层负责把具体硬件和抽象接口绑定起来。

示例：

```c
static stm32_uart_priv_t uart1_priv = {
    .instance = USART1,
    .baudrate = 115200,
};

uart_t board_console_uart = {
    .ops = &stm32_uart_ops,
    .priv = &uart1_priv,
};
```

业务层只引用：

```c
extern uart_t board_console_uart;
```

或者通过系统上下文传入：

```c
app_context_t app_ctx = {
    .log_uart = &board_console_uart,
};
```

## 十七、编码检查清单

### 17.1 应该这样做

-  上层只调用抽象接口
    
-  寄存器访问只出现在 BSP、HAL 或 driver 层
    
-  每类外设定义统一 ops 函数表
    
-  每个设备对象包含 `ops` 和 `priv`
    
-  `priv` 保存硬件上下文
    
-  提供接口包装函数，不让上层散落调用 `ops`
    
-  错误码统一
    
-  初始化和反初始化流程明确
    
-  支持 mock 或 fake 实现
    
-  接口命名稳定、清晰、平台无关

### 17.2 不应该这样做

-  业务层直接访问寄存器
    
-  业务层直接调用芯片厂商 HAL
    
-  全局硬件对象在各处被随意引用
    
-  宏中隐藏复杂硬件操作
    
-  接口函数名带具体芯片型号
    
-  不同驱动返回不同错误码风格
    
-  `void *priv` 无校验、无文档说明
    
-  ops 函数表运行时被随意修改

## 十八、命名规范建议

### 18.1 抽象接口

```c
uart_init()
uart_send()
uart_recv()
gpio_set()
gpio_get()
spi_transfer()
i2c_write()
i2c_read()
```

### 18.2 具体实现

```c
stm32_uart_init()
stm32_uart_send()
esp32_uart_send()
mock_uart_send()
```

### 18.3 ops 表

```c
stm32_uart_ops
esp32_uart_ops
mock_uart_ops
```

### 18.4 私有数据

```c
stm32_uart_priv_t
esp32_uart_priv_t
mock_uart_priv_t
```

## 十九、注意事项

### 19.1 函数指针开销

函数指针会带来间接调用开销。

在以下场景应谨慎使用：

- 中断极短路径
    
- 高频控制环
    
- 极致性能路径
    
- 资源非常紧张的 MCU

但在大多数驱动抽象、协议栈、设备管理、业务模块中，该开销通常可以接受。

### 19.2 不要过度抽象

不是所有模块都需要 ops 表。

适合使用接口抽象的场景：

- 硬件平台可能变化
    
- 外设有多个实现
    
- 需要 mock 测试
    
- 驱动需要复用
    
- 项目生命周期较长
    
- 多型号产品共用代码

不适合过度抽象的场景：

- 一次性小项目
    
- 只有单一固定硬件
    
- 性能极其敏感
    
- 抽象层比业务本身还复杂

## 二十、推荐模板

### 20.1 接口头文件模板

```c
#ifndef DEVICE_IF_H
#define DEVICE_IF_H

typedef struct device device_t;

typedef struct {
    int (*init)(device_t *dev);
    int (*deinit)(device_t *dev);
    int (*read)(device_t *dev, void *buf, unsigned int len);
    int (*write)(device_t *dev, const void *buf, unsigned int len);
} device_ops_t;

struct device {
    const device_ops_t *ops;
    void *priv;
};

int device_init(device_t *dev);
int device_deinit(device_t *dev);
int device_read(device_t *dev, void *buf, unsigned int len);
int device_write(device_t *dev, const void *buf, unsigned int len);

#endif
```

### 20.2 接口实现模板

```c
int device_write(device_t *dev, const void *buf, unsigned int len)
{
    if (dev == NULL || dev->ops == NULL || dev->ops->write == NULL) {
        return -1;
    }

    return dev->ops->write(dev, buf, len);
}
```

### 20.3 具体驱动模板

```c
typedef struct {
    void *hw_base;
    unsigned int irq;
} xxx_device_priv_t;

static int xxx_device_write(device_t *dev, const void *buf, unsigned int len)
{
    xxx_device_priv_t *priv = (xxx_device_priv_t *)dev->priv;

    if (priv == NULL || buf == NULL) {
        return -1;
    }

    /*
     * Hardware-specific operation here.
     */

    return 0;
}

const device_ops_t xxx_device_ops = {
    .init = xxx_device_init,
    .deinit = xxx_device_deinit,
    .read = xxx_device_read,
    .write = xxx_device_write,
};
```

## 二十一、总结

嵌入式 C 中使用 **“结构体 + 函数指针表 + 私有上下文”** 是一种成熟的硬件解耦方式。

其核心原则是：

```text
业务逻辑依赖抽象接口
抽象接口屏蔽硬件差异
具体驱动封装硬件细节
BSP 负责对象装配
```

最终目标是：

- 换硬件不改业务
    
- 换驱动不改接口
    
- 上层代码可测试
    
- 底层实现可替换
    
- 项目结构清晰可维护