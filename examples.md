# esp32GPIO使用示例

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

## 2.KEY控制LED

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
    io_conf.mode = GPIO_MODE_OUTPUT; // 设置为输出模式
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

void app_main(void)
{
    // 初始化LED
    led_init();

    while (1) 
    {
        led_toggle(); // 切换LED状态
        vTaskDelay(pdMS_TO_TICKS(1000)); // 延时1秒
    }
}
```

## 