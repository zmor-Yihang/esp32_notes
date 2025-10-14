# ESP32 GPTimer API 完整函数参考

## 0. GPTimer主要数据结构

### gptimer_config_t

通用定时器配置结构体，用于创建新的GPTimer实例时的初始化配置。

**结构体定义：**
```c
typedef struct {
    gptimer_clock_source_t clk_src;     // 时钟源选择
    gptimer_count_direction_t direction; // 计数方向
    uint32_t resolution_hz;              // 计数器分辨率(Hz)
    int intr_priority;                   // 中断优先级
    struct {
        uint32_t intr_shared: 1;         // 中断共享标志
        uint32_t allow_pd: 1;            // 允许电源域关闭
        uint32_t backup_before_sleep: 1; // 睡眠前备份(已弃用)
    } flags;
} gptimer_config_t;
```

**字段详细说明：**

- **`clk_src`**：GPTimer时钟源选择
  - `GPTIMER_CLK_SRC_APB`：APB时钟源(80MHz)，精度高，功耗相对较大
  - `GPTIMER_CLK_SRC_XTAL`：晶振时钟源(40MHz)，精度中等，功耗中等
  - `GPTIMER_CLK_SRC_RC_FAST`：内部RC时钟源(约17.5MHz)，精度低，功耗最小

- **`direction`**：计数方向
  - `GPTIMER_COUNT_UP`：向上计数(0→最大值)
  - `GPTIMER_COUNT_DOWN`：向下计数(最大值→0)

- **`resolution_hz`**：计数器分辨率(工作频率，Hz)
  - 取值范围：1Hz ~ 时钟源频率
  - 每个计数tick的时间步长 = 1/resolution_hz 秒
  - 示例：resolution_hz=1000000时，每tick=1微秒

- **`intr_priority`**：GPTimer中断优先级
  - 取值范围：0~7 (ESP32) 或 0~15 (ESP32-S2/S3/C3等)
  - 设置为0时：驱动程序自动分配相对低优先级的中断(1,2,3)
  - 数值越大优先级越高

- **`flags.intr_shared`**：中断共享标志
  - true：允许定时器中断号与其他外设共享，可节省中断资源
  - false：独占中断号，响应延迟更低

- **`flags.allow_pd`**：电源域管理
  - true：系统进入睡眠模式时，允许GPTimer电源域被关闭
  - false：睡眠时保持GPTimer电源域开启

- **`flags.backup_before_sleep`**：睡眠前备份(已弃用)
  - 与allow_pd含义相同，为向后兼容保留

**使用示例：**
```c
gptimer_config_t timer_config = {
    .clk_src = GPTIMER_CLK_SRC_APB,      // 使用APB时钟源
    .direction = GPTIMER_COUNT_UP,        // 向上计数
    .resolution_hz = 1000000,             // 1MHz分辨率，1us per tick
    .intr_priority = 0,                   // 自动分配中断优先级
    .flags = {
        .intr_shared = false,             // 独占中断
        .allow_pd = false,                // 不允许电源域关闭
    },
};
```

### gptimer_alarm_config_t

通用定时器告警配置结构体，用于设置定时器的告警触发条件和行为。

**结构体定义：**
```c
typedef struct {
    uint64_t alarm_count;               // 告警目标计数值
    uint64_t reload_count;              // 告警重载计数值
    struct {
        uint32_t auto_reload_on_alarm: 1; // 告警时自动重载标志
    } flags;
} gptimer_alarm_config_t;
```

**字段详细说明：**

- **`alarm_count`**：告警目标计数值
  - 当计数器达到此值时触发告警事件
  - 取值范围：0 ~ 2^64-1
  - 对于向上计数：告警值应大于当前计数值
  - 对于向下计数：告警值应小于当前计数值

- **`reload_count`**：告警重载计数值
  - 仅当auto_reload_on_alarm=true时生效
  - 告警触发后，计数器自动重载到此值继续计数
  - 实现周期性定时器的关键参数

- **`flags.auto_reload_on_alarm`**：告警时自动重载标志
  - true：告警触发时硬件自动将计数器重载为reload_count值
  - false：告警触发后计数器继续按原方向计数

**使用模式：**

1. **单次告警模式**：
```c
gptimer_alarm_config_t alarm_config = {
    .alarm_count = 1000000,              // 1秒后告警(假设1MHz分辨率)
    .flags.auto_reload_on_alarm = false, // 不自动重载
};
```

2. **周期性告警模式**：
```c
gptimer_alarm_config_t alarm_config = {
    .alarm_count = 1000000,              // 1秒周期
    .reload_count = 0,                   // 重载到0重新开始计数
    .flags.auto_reload_on_alarm = true,  // 自动重载实现周期性
};
```

### gptimer_event_callbacks_t

支持的GPTimer回调函数组结构体，用于注册各种事件的回调函数。

**结构体定义：**
```c
typedef struct {
    gptimer_alarm_cb_t on_alarm;        // 定时器告警回调函数
} gptimer_event_callbacks_t;
```

**字段详细说明：**

- **`on_alarm`**：定时器告警回调函数指针
  - 当定时器告警事件发生时被调用
  - 在中断上下文中执行，需要遵循ISR编程规范
  - 设置为NULL表示不需要告警回调

**使用示例：**
```c
static bool IRAM_ATTR timer_alarm_cb(gptimer_handle_t timer, 
                                     const gptimer_alarm_event_data_t *edata, 
                                     void *user_ctx) {
    BaseType_t high_task_awoken = pdFALSE;
    // 处理告警事件
    return high_task_awoken == pdTRUE;
}

gptimer_event_callbacks_t cbs = {
    .on_alarm = timer_alarm_cb,
};
```

### gptimer_etm_event_config_t

GPTimer ETM(Event Task Matrix)事件配置结构体，用于配置定时器产生的ETM事件。

**结构体定义：**
```c
typedef struct {
    gptimer_etm_event_type_t event_type; // ETM事件类型
} gptimer_etm_event_config_t;
```

**字段详细说明：**

- **`event_type`**：GPTimer ETM事件类型
  - `GPTIMER_ETM_EVENT_ALARM`：告警事件，当定时器告警触发时产生ETM事件
  - 可以与其他外设的ETM任务连接，实现硬件级别的自动触发

**ETM功能说明：**
ETM(Event Task Matrix)是ESP32系列芯片的硬件事件连接矩阵，允许外设之间在硬件层面直接通信，无需CPU介入。GPTimer可以作为事件源，触发其他外设的任务。

### gptimer_etm_task_config_t

GPTimer ETM任务配置结构体，用于配置定时器接收的ETM任务。

**结构体定义：**
```c
typedef struct {
    gptimer_etm_task_type_t task_type;   // ETM任务类型
} gptimer_etm_task_config_t;
```

**字段详细说明：**

- **`task_type`**：GPTimer ETM任务类型
  - `GPTIMER_ETM_TASK_START_COUNT`：启动计数任务
  - `GPTIMER_ETM_TASK_STOP_COUNT`：停止计数任务
  - `GPTIMER_ETM_TASK_EN_ALARM`：使能告警任务
  - `GPTIMER_ETM_TASK_RELOAD`：重载计数器任务
  - `GPTIMER_ETM_TASK_CAPTURE`：捕获计数值任务

### gptimer_alarm_event_data_t

定时器告警事件数据结构体，在告警回调函数中提供事件相关的数据信息。

**结构体定义：**
```c
typedef struct {
    uint64_t count_value;               // 当前计数值
    uint64_t alarm_value;               // 当前告警值
} gptimer_alarm_event_data_t;
```

**字段详细说明：**

- **`count_value`**：触发告警时的实际计数值
  - 通常等于alarm_value，但在高频率下可能略有差异
  - 可用于精确的时间戳计算

- **`alarm_value`**：触发此次告警的目标值
  - 与gptimer_alarm_config_t中设置的alarm_count相同
  - 用于确认告警来源

**使用示例：**
```c
static bool IRAM_ATTR timer_alarm_cb(gptimer_handle_t timer, 
                                     const gptimer_alarm_event_data_t *edata, 
                                     void *user_ctx) {
    printf("告警触发！计数值=%llu, 告警值=%llu\n", 
           edata->count_value, edata->alarm_value);
    return false;
}
```

### gptimer_alarm_cb_t

定时器告警回调函数类型定义，规定了告警回调函数的原型和调用约定。

**函数原型：**
```c
typedef bool (*gptimer_alarm_cb_t)(gptimer_handle_t timer, 
                                   const gptimer_alarm_event_data_t *edata, 
                                   void *user_ctx);
```

**参数详细说明：**

- **`timer`**：触发告警的定时器句柄
  - 由gptimer_new_timer创建的定时器句柄
  - 可用于在回调中操作定时器（如重新设置告警）

- **`edata`**：告警事件数据
  - 包含告警触发时的计数值和告警值
  - 由驱动程序自动填充，只读

- **`user_ctx`**：用户自定义数据指针
  - 从gptimer_register_event_callbacks传递的用户数据
  - 可以是任何类型的指针，用于传递上下文信息

**返回值说明：**
- **true**：表示有高优先级任务被此回调函数唤醒，需要进行任务切换
- **false**：无需任务切换，继续执行当前任务

**重要注意事项：**

1. **ISR上下文**：回调函数在中断服务程序(ISR)上下文中执行
2. **IRAM属性**：建议使用IRAM_ATTR属性确保代码在内存中
3. **执行时间**：应尽可能短，避免长时间占用中断
4. **API限制**：只能调用支持ISR上下文的API函数
5. **数据访问**：避免访问flash中的数据，优先使用RAM中的数据

**回调函数示例：**
```c
// 简单的LED闪烁回调
static bool IRAM_ATTR led_blink_cb(gptimer_handle_t timer, 
                                   const gptimer_alarm_event_data_t *edata, 
                                   void *user_ctx) {
    gpio_num_t led_pin = *(gpio_num_t*)user_ctx;
    static bool led_state = false;
    
    led_state = !led_state;
    gpio_set_level(led_pin, led_state);
    
    return false; // 无需任务切换
}

// 任务通知回调
static bool IRAM_ATTR task_notify_cb(gptimer_handle_t timer, 
                                     const gptimer_alarm_event_data_t *edata, 
                                     void *user_ctx) {
    TaskHandle_t task_handle = (TaskHandle_t)user_ctx;
    BaseType_t high_task_awoken = pdFALSE;
    
    vTaskNotifyGiveFromISR(task_handle, &high_task_awoken);
    
    return high_task_awoken == pdTRUE; // 根据是否有高优先级任务被唤醒返回
}
```

## 1. GPTimer初始化和配置函数

### gptimer_new_timer
**函数原型：**
```c
esp_err_t gptimer_new_timer(const gptimer_config_t *config, gptimer_handle_t *ret_timer);
```
**参数：**
config：指向包含GPTimer配置信息的gptimer_config_t结构体的指针
ret_timer：返回的定时器句柄指针
**返回值：**
ESP_OK：创建GPTimer成功
ESP_ERR_INVALID_ARG：参数错误导致创建失败
ESP_ERR_NO_MEM：内存不足导致创建失败
ESP_ERR_NOT_FOUND：所有硬件定时器都被使用，没有可用的定时器
ESP_FAIL：其他错误导致创建失败
**作用：**创建一个新的通用定时器，并返回句柄。新创建的定时器处于"init"状态

### gptimer_del_timer
**函数原型：**
```c
esp_err_t gptimer_del_timer(gptimer_handle_t timer);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
**返回值：**
ESP_OK：删除GPTimer成功
ESP_ERR_INVALID_ARG：参数错误导致删除失败
ESP_ERR_INVALID_STATE：定时器不在init状态导致删除失败
ESP_FAIL：其他错误导致删除失败
**作用：**删除GPTimer句柄。定时器必须处于"init"状态才能被删除

### gptimer_get_resolution
**函数原型：**
```c
esp_err_t gptimer_get_resolution(gptimer_handle_t timer, uint32_t *out_resolution);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
out_resolution：返回的定时器分辨率(Hz)
**返回值：**
ESP_OK：获取GPTimer分辨率成功
ESP_ERR_INVALID_ARG：参数错误导致获取失败
ESP_FAIL：其他错误导致获取失败
**作用：**返回定时器的实际分辨率。通常与配置的分辨率相同，但某些不稳定的时钟源(如RC_FAST)会进行校准，实际分辨率可能不同

## 2. GPTimer状态控制函数

### gptimer_enable
**函数原型：**
```c
esp_err_t gptimer_enable(gptimer_handle_t timer);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
**返回值：**
ESP_OK：使能GPTimer成功
ESP_ERR_INVALID_ARG：参数错误导致使能失败
ESP_ERR_INVALID_STATE：定时器已经使能导致使能失败
ESP_FAIL：其他错误导致使能失败
**作用：**使能GPTimer，将定时器状态从"init"转换为"enable"。会启用中断服务(如果已注册)并获取PM锁

### gptimer_disable
**函数原型：**
```c
esp_err_t gptimer_disable(gptimer_handle_t timer);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
**返回值：**
ESP_OK：禁用GPTimer成功
ESP_ERR_INVALID_ARG：参数错误导致禁用失败
ESP_ERR_INVALID_STATE：定时器未使能导致禁用失败
ESP_FAIL：其他错误导致禁用失败
**作用：**禁用GPTimer，将定时器状态从"enable"转换为"init"。会禁用中断服务并释放PM锁

### gptimer_start
**函数原型：**
```c
esp_err_t gptimer_start(gptimer_handle_t timer);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
**返回值：**
ESP_OK：启动GPTimer成功
ESP_ERR_INVALID_ARG：参数错误导致启动失败
ESP_ERR_INVALID_STATE：定时器未使能或已在运行导致启动失败
ESP_FAIL：其他错误导致启动失败
**作用：**启动GPTimer(内部计数器开始计数)，将定时器状态从"enable"转换为"run"

### gptimer_stop
**函数原型：**
```c
esp_err_t gptimer_stop(gptimer_handle_t timer);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
**返回值：**
ESP_OK：停止GPTimer成功
ESP_ERR_INVALID_ARG：参数错误导致停止失败
ESP_ERR_INVALID_STATE：定时器未在运行导致停止失败
ESP_FAIL：其他错误导致停止失败
**作用：**停止GPTimer(内部计数器停止计数)，将定时器状态从"run"转换为"enable"

## 3. GPTimer计数值控制函数

### gptimer_set_raw_count
**函数原型：**
```c
esp_err_t gptimer_set_raw_count(gptimer_handle_t timer, uint64_t value);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
value：要设置的计数值
**返回值：**
ESP_OK：设置GPTimer原始计数值成功
ESP_ERR_INVALID_ARG：参数错误导致设置失败
ESP_FAIL：其他错误导致设置失败
**作用：**设置GPTimer原始计数值。当更新活动定时器的原始计数时，定时器会立即从新值开始计数

### gptimer_get_raw_count
**函数原型：**
```c
esp_err_t gptimer_get_raw_count(gptimer_handle_t timer, uint64_t *value);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
value：返回的GPTimer计数值指针
**返回值：**
ESP_OK：获取GPTimer原始计数值成功
ESP_ERR_INVALID_ARG：参数错误导致获取失败
ESP_FAIL：其他错误导致获取失败
**作用：**获取GPTimer原始计数值。会触发软件捕获事件然后返回捕获的计数值

### gptimer_get_captured_count
**函数原型：**
```c
esp_err_t gptimer_get_captured_count(gptimer_handle_t timer, uint64_t *value);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
value：返回的捕获计数值指针
**返回值：**
ESP_OK：获取GPTimer捕获计数值成功
ESP_ERR_INVALID_ARG：参数错误导致获取失败
ESP_FAIL：其他错误导致获取失败
**作用：**获取GPTimer捕获的计数值。不会触发软件捕获事件，只返回最后捕获的计数值

## 4. GPTimer告警配置函数

### gptimer_set_alarm_action
**函数原型：**
```c
esp_err_t gptimer_set_alarm_action(gptimer_handle_t timer, const gptimer_alarm_config_t *config);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
config：告警配置，设置为NULL表示禁用告警功能
**返回值：**
ESP_OK：为GPTimer设置告警动作成功
ESP_ERR_INVALID_ARG：参数错误导致设置失败
ESP_FAIL：其他错误导致设置失败
**作用：**为GPTimer设置告警事件动作。可以在ISR上下文中运行，允许在任何ISR回调中立即更新新的告警动作

## 5. GPTimer回调函数管理

### gptimer_register_event_callbacks
**函数原型：**
```c
esp_err_t gptimer_register_event_callbacks(gptimer_handle_t timer, const gptimer_event_callbacks_t *cbs, void *user_data);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
cbs：回调函数组
user_data：用户数据，将直接传递给回调函数
**返回值：**
ESP_OK：设置事件回调成功
ESP_ERR_INVALID_ARG：参数错误导致设置失败
ESP_ERR_INVALID_STATE：定时器不在init状态导致设置失败
ESP_FAIL：其他错误导致设置失败
**作用：**为GPTimer设置回调函数。用户注册的回调函数预期在ISR上下文中运行

## 6. GPTimer ETM功能函数

### gptimer_new_etm_event
**函数原型：**
```c
esp_err_t gptimer_new_etm_event(gptimer_handle_t timer, const gptimer_etm_event_config_t *config, esp_etm_event_handle_t *out_event);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
config：GPTimer ETM事件配置
out_event：返回的ETM事件句柄
**返回值：**
ESP_OK：获取ETM事件成功
ESP_ERR_INVALID_ARG：参数错误导致获取失败
ESP_FAIL：其他错误导致获取失败
**作用：**为GPTimer获取ETM事件。创建的ETM事件对象可以稍后通过调用esp_etm_del_event删除

### gptimer_new_etm_task
**函数原型：**
```c
esp_err_t gptimer_new_etm_task(gptimer_handle_t timer, const gptimer_etm_task_config_t *config, esp_etm_task_handle_t *out_task);
```
**参数：**
timer：由gptimer_new_timer创建的定时器句柄
config：GPTimer ETM任务配置
out_task：返回的ETM任务句柄
**返回值：**
ESP_OK：获取ETM任务成功
ESP_ERR_INVALID_ARG：参数错误导致获取失败
ESP_FAIL：其他错误导致获取失败
**作用：**为GPTimer获取ETM任务。创建的ETM任务对象可以稍后通过调用esp_etm_del_task删除

---

