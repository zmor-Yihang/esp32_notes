# esp32 GPIO使用示例

## 1.LED闪烁

**注意**：mode如果设置为输出模式，gpio_set_level函数不能读取到正确的电平，led_toggle就会有问题

**纯输出模式（`GPIO_MODE_OUTPUT`）下，ESP32 的输入缓冲器默认是关闭的，所以 `gpio_get_level()` 读取的是“输出锁存器”的值，而不是引脚上的真实电平。如果外部电路强行拉低/拉高引脚，你读不到真实状态！**

[输出驱动器] ←─ 输出锁存器（用户设置的值）
     │
     └─── 引脚 ────┐
                   │
           [输入缓冲器] ←─ 读取引脚真实电平

> io_conf.mode = GPIO_MODE_INPUT_OUTPUT; // 设置为输入输出模式

```c
/* led.c文件 */
#include "led.h"

/**
 * @brief LED初始化
 * @param None
 * @retval None
 */
void led_init(void)
{
    gpio_config_t io_conf = {0};

    io_conf.intr_type = GPIO_INTR_DISABLE; // 禁用中断
    io_conf.mode = GPIO_MODE_INPUT_OUTPUT; // 设置为输入输出模式
    io_conf.pin_bit_mask = (1ULL << LED_PIN); // 配置LED引脚
    io_conf.pull_down_en = GPIO_PULLDOWN_DISABLE; // 禁用下拉
    io_conf.pull_up_en = GPIO_PULLUP_DISABLE; // 禁用上拉
    gpio_config(&io_conf); // 初始化

    led_off(); // 初始化时关闭LED
}

/**
 * @brief LED开启
 * @param None
 * @retval None
 */
void led_on(void)
{
    gpio_set_level(LED_PIN, LED_ON);
}

/**
 * @brief LED关闭
 * @param None
 * @retval None
 */
void led_off(void)
{
    gpio_set_level(LED_PIN, LED_OFF);
}

/**
 * @brief LED状态切换
 * @param None
 * @retval None
 */
void led_toggle(void)
{
    gpio_set_level(LED_PIN, !gpio_get_level(LED_PIN));
}
```

```c
/* led.h文件 */
#ifndef __LED_H__
#define __LED_H__

#include "driver/gpio.h"

// LED引脚定义
#define LED_PIN GPIO_NUM_1

// LED状态定义
#define LED_ON  1
#define LED_OFF 0

void led_init(void);

void led_on(void);

void led_off(void);

void led_toggle(void);

#endif /* __LED_H__ */
```

```c
/* main.c文件 */
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "led.h"
#include "driver/gpio.h"

void app_main(void)
{
    led_init();
    
    while (1) 
    {
        led_toggle(); // 切换LED状态
        int current_level = gpio_get_level(GPIO_NUM_1);
        printf("LED Toggled - GPIO1 Level: %d\n", current_level);
        vTaskDelay(pdMS_TO_TICKS(1000)); // 延时1秒
    }
}
```

## 2.KEY中断控制LED

**esp32中断机制: 软件分发+共享中断源**

- **共享中断向量（Shared IRQ）**
  ESP32 的所有 GPIO 中断共用少数几个硬件中断源（如 PRO_CPU 的 GPIO_INT0~GPIO_INT4）。
  → 硬件无法直接区分是哪个引脚触发的中断。
- **统一中断服务入口**
  `gpio_install_isr_service(0)` 安装的是一个**系统级中断分发器**，它：
  - 接收所有 GPIO 中断；
  - 查询状态寄存器，判断触发引脚；
  - 调用你“注册”的对应处理函数。
- **动态注册回调函数**
  `gpio_isr_handler_add(pin, handler, arg)` 将你的中断处理函数注册到系统内部的“回调表”中。
  → 中断发生时，系统查表调用对应 handler。

```c
/* key.c文件 */
#include "key.h"

/* 按键中断标志 */
volatile bool FlagOfButtonPressed = false;

/**
 * @brief GPIO中断服务程序, 仅设置中断标志位
 * 
 * @param arg 
 */
static void IRAM_ATTR gpio_isr_handler(void* arg)
{
    FlagOfButtonPressed = true;  // 置中断标志
}

/**
 * @brief 按键初始化
 * @note  配置KEY0引脚为输入模式, 上拉, 下降沿中断，
 * 
 */
void key_init(void)
{
    gpio_config_t io_conf = {0};

    io_conf.pin_bit_mask = (1ULL << KEY_BOOT);    // 配置KEY0引脚
    io_conf.mode = GPIO_MODE_INPUT;               // 设置为输入模式
    io_conf.pull_up_en = GPIO_PULLUP_ENABLE;      // 启用上拉
    io_conf.pull_down_en = GPIO_PULLDOWN_DISABLE; // 禁用下拉
    io_conf.intr_type = GPIO_INTR_NEGEDGE;        // 启用中断
    gpio_config(&io_conf);                        // 初始化GPIO

    gpio_install_isr_service(0);                  // 安装GPIO中断服务, 相当于使能NVIC [HAL_NVIC_EnableIRQ(EXTI0_IRQn)]

    gpio_isr_handler_add(KEY_BOOT, gpio_isr_handler, (void *)KEY_BOOT); // 注册中断处理函数
}

/**
 * @brief 获取按键状态
 * @note  按键消抖在主循环中处理
 * @return uint8_t 按键状态, 0表示按下, 1表示未按下
 */
uint8_t key_get_state(void)
{
    return gpio_get_level(KEY_BOOT);
}
```

```c
/* key.h文件 */
#ifndef __KEY_H__
#define __KEY_H__

#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define KEY_BOOT GPIO_NUM_0

extern volatile bool FlagOfButtonPressed;

void key_init(void);
uint8_t key_get_state(void);

#endif // __KEY_H__
```

```c
/* main.c文件 */
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "led.h"
#include "key.h"

void app_main(void)
{
    // 初始化LED
    led_init();
    
    // 初始化按键
    key_init();

    while (1) 
    {
        if (FlagOfButtonPressed) 
        {
            vTaskDelay(pdMS_TO_TICKS(50)); // 简单消抖
            if (key_get_state() == 0) // 确认按键仍然按下
            {
                while (key_get_state() == 0); // 等待按键释放
                led_toggle();                 // 切换LED状态
                FlagOfButtonPressed = false;  // 清除按键标志
            }
            /* 误触 */
            FlagOfButtonPressed = false; // 如果按键已释放，仍然清除标志
        }
    }
}
```

# esp32 UART使用示例

## 1.UART串口阻塞收发

```c
/* uart.c文件 */
#include "uart.h"

/* UART接收缓冲区 */
uint8_t bufferOfUartReceive[lengthOfUartReceive] = {0};

/* UART发送缓冲区 */
uint8_t bufferOfUartSend[lengthOfUartSend] = {0};


/**
 * @brief UART串口初始化函数
 * @details 配置并初始化UART0串口，设置通信参数和引脚映射
 * @note 函数内部会自动分配内存创建环形缓冲区
 */
void uart_init(void) /* UART串口初始化函数 */
{
    uart_config_t uart_config = {0}; /* 创建UART配置结构体并初始化为0 */

    uart_config.baud_rate = 115200;                   /* 设置波特率为115200 */
    uart_config.data_bits = UART_DATA_8_BITS;         /* 设置数据位为8位 */
    uart_config.stop_bits = UART_STOP_BITS_1;         /* 设置停止位为1位 */
    uart_config.parity = UART_PARITY_DISABLE;         /* 设置校验位：无校验 */
    uart_config.source_clk = UART_SCLK_APB;           /* 设置时钟源为APB时钟 */
    uart_config.flow_ctrl = UART_HW_FLOWCTRL_DISABLE; /* 关闭硬件流控 */
    uart_config.rx_flow_ctrl_thresh = 0;              /* 设置接收流控阈值（流控关闭时不生效） */

    uart_param_config(UART_NUM_0, &uart_config); /* 将配置结构体参数传入UART0初始化函数 */

    /* 设置UART0的引脚 */
    uart_set_pin(UART_NUM_0, GPIO_NUM_43, GPIO_NUM_44, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);

    /* 该函数内部会自动调用 heap_caps_malloc() 等函数分配内存，创建环形缓冲区 */
    uart_driver_install(UART_NUM_0, 512, 512, 20, NULL, 0);
}

/**
 * @brief UART发送字节函数
 * @details 通过UART0端口发送指定缓冲区中的数据
 * 
 * @param buffer 指向要发送数据的缓冲区指针，数据必须以'\0'结尾
 * @note 发送的数据长度由strlen()函数计算得出
 * @note 函数会阻塞直到所有数据发送完成
 */
void uart_send_bytes(uint8_t *buffer) /* UART发送函数 */
{
    uart_write_bytes(UART_NUM_0, (char *)buffer, strlen((char *)buffer));
}

/**
 * @brief UART接收字节函数
 * @param[out] buffer 指向接收数据缓冲区的指针，用于存储接收到的数据
 * @return int 实际接收到的字节数
 * @note 接收数据之前会清空接收缓冲区
 * @note 函数会自动在接收到的数据末尾添加字符串结束符'\0'
 */
int uart_receive_bytes(uint8_t *buffer) /* UART接收函数 */
{
    memset(bufferOfUartReceive, 0, lengthOfUartReceive);       /* 清空接收缓冲区，全部填充为0 */
    int bytes_read = uart_read_bytes(UART_NUM_0, (char *)buffer, lengthOfUartReceive - 1, 1000);

    if (bytes_read > 0)
    {
        buffer[bytes_read] = '\0'; // 添加字符串结束符
    }
    return bytes_read;
}
```

```c
/* uart.h文件 */
#ifndef __UART_H__
#define __UART_H__

#include "driver/uart.h"
#include "driver/gpio.h"
#include <string.h>

/* UART接收缓冲区长度 */
#define lengthOfUartReceive 64

/* UART发送缓冲区长度 */
#define lengthOfUartSend 64

/* UART接收缓冲区 */
extern uint8_t bufferOfUartReceive[lengthOfUartReceive];

/* UART发送缓冲区 */
extern uint8_t bufferOfUartSend[lengthOfUartSend];

void uart_init(void);

void uart_send_bytes(uint8_t *sendBuffer);

int uart_receive_bytes(uint8_t *receiveBuffer);

#endif /* __UART_H__ */

```

```c
/* main.c文件 */
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "led.h"
#include "uart.h"
#include <string.h>

void app_main(void)
{
    led_init();
    uart_init();

    bufferOfUartSend[0] = 'A';
    bufferOfUartSend[1] = 'B';
    bufferOfUartSend[2] = '\0';

    while (1)
    {
        uart_send_bytes(bufferOfUartSend); /* 发送字符串"AB"到串口 */

        uart_receive_bytes(bufferOfUartReceive); /* 从串口接收数据到缓冲区 */

        printf("\n%s\n", bufferOfUartReceive); /* 打印接收到的数据 */

        printf("hello world\n"); /* 打印调试信息 */
        led_toggle();            /* 切换LED状态 */
        vTaskDelay(1000);        /* 延时1000ms，控制循环执行频率 */
    }
}
```

# esp32 定时器使用示例

## 1.定时中断翻转LED (System Timer)

系统定时器一般只用于定时功能，只需要定时中断函数，**API在 esp_timer.h 声明，属于系统内核**

```c
/* systimer.c文件 */
#include "systimer.h"

/* 系统定时器句柄 */
esp_timer_handle_t systimer_handle = NULL;

/**
 * @brief 定时器回调函数
 * @details 当定时器到期时调用此函数。
 * @param arg 指向传入数据的指针
 */
static void systimer_callback(void* arg)
{
    // Timer callback function
    printf("Timer callback executed\n");
    led_toggle();
}

/**
 * @brief 初始化系统定时器
 * @details 定时器配置为在任务上下文中运行，不跳过未处理的事件。
 */
void systimer_init(void)
{
    esp_timer_create_args_t create_args = {0};

    create_args.callback = systimer_callback;
    create_args.arg = NULL;
    create_args.name = "systimer";
    create_args.dispatch_method = ESP_TIMER_TASK;
    create_args.skip_unhandled_events = false;

    esp_timer_create(&create_args, &systimer_handle);
}

/**
 * @brief 启动定时器进行一次执行
 * @param timeout 超时时间（毫秒）
 */
void systimer_start_once(int timeout)
{
    esp_timer_start_once(systimer_handle, timeout * 1000);
}

/**
 * @brief 启动定时器进行周期性执行
 * @param period 定时器执行间隔（毫秒）
 */
void systimer_start_period(int period)
{
    esp_timer_start_periodic(systimer_handle, period * 1000);
}

/**
 * @brief 停止系统定时器
 */
void systimer_stop(void)
{
    esp_timer_stop(systimer_handle);
}
```

```c
/* systimer.h文件 */
#ifndef __SYSTIMER_H__
#define __SYSTIMER_H__

#include "esp_timer.h"
#include "led.h"

extern esp_timer_handle_t systimer_handle;

void systimer_init(void);
void systimer_start_once(int timeout);
void systimer_start_period(int period);
void systimer_stop(void);


#endif // __SYSTIMER_H__
```

```c
/* main文件 */
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "systimer.h"

void app_main(void)
{
    // queue_init();
    // task_init();
    led_init();
    systimer_init();
    systimer_start_period(1000); // 启动周期性定时器

    vTaskDelete(NULL); // 删除主任务main
}
```

## 2.PWM控制LED (General Timer)
