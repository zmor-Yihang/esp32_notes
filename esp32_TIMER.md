# ESP32 Timer API 完整函数参考

## 0. Timer数据结构

### esp_timer_create_args_t

**结构体定义：**

```c
typedef struct {
    esp_timer_cb_t callback;                    // 定时器到期时跳转到该回调函数
    void* arg;                                  // 传递给回调函数的参数, 可以借助该参数在一个回调函数中处理多个定时器到期任务
    esp_timer_dispatch_t dispatch_method;       // 从任务或ISR分发回调；如果未指定，使用esp_timer任务
    const char* name;                           // 定时器名称，在esp_timer_dump()函数中使用
    bool skip_unhandled_events;                 // 在轻度睡眠中跳过周期性定时器未处理事件的设置
} esp_timer_create_args_t;

typedef struct {
    /**
     * @brief 定时器到期时执行的回调函数指针
     * 当定时器时间到达时，系统将调用此函数。
     * 函数原型必须为：void callback(void* arg)
     */
    esp_timer_cb_t callback;

    /**
     * @brief 传递给回调函数的用户自定义参数
     *
     * 类型为 void*，允许你传入任意类型的数据（如结构体指针、设备句柄、计数器等）。
     * 此参数在回调函数中通过 void* arg 接收，可实现“一个回调函数处理多个定时器”的复用逻辑。
     */
    void* arg;

    /**
     * @brief 回调函数的分发方式（执行上下文）
     *
     * 选ESP_TIMER_TASK时，在定时中断中使用队列发送信号给任务 "esp_timer"，回调函数在该任务中执行
     * 选ESP_TIMER_ISR时，直接在定时中断中执行回调函数
     * 
     * 除非明确需要微秒级确定性延迟，否则一律使用 ESP_TIMER_TASK。
     */
    esp_timer_dispatch_t dispatch_method;

    /**
     * @brief 定时器名称（调试用）
     *
     * 此字符串用于调试和诊断，当调用 esp_timer_dump() 时会打印出来。
     * 帮助开发者识别“哪个定时器在运行、周期是多少、下次何时触发”。
     * 可设为 NULL，但建议填写有意义的名称，便于调试。
     */
    const char* name;

    /**
     * @brief 在轻度睡眠（Light Sleep）期间是否跳过未处理的周期性定时器事件
     *
     * 此字段仅对周期性定时器（esp_timer_start_periodic）生效，对单次定时器无影响。
     *
     * 背景：
     * 当系统进入 Light Sleep 时，CPU 停止运行，但 SYSTIMER 仍在计数。
     * 如果定时器周期较短（如 100ms），在睡眠 1 秒后唤醒，可能“积压”10 个到期事件。
     *
     * 行为：
     * - false（默认）→ 系统唤醒后，依次执行所有积压的回调（可能导致“回调风暴”，阻塞系统）
     * - true → 系统唤醒后，只执行“最新一次”回调，跳过中间所有未处理事件
     *
     * 适用场景：
     * - true → 用于“心跳”、“状态刷新”、“UI 重绘”等可跳过中间状态的任务
     *          例：LED 闪烁、传感器轮询、屏幕刷新 —— 丢失几次无所谓
     * - false → 用于“计数器”、“事件记录”、“数据采样”等不能丢失任何触发的任务
     *          例：脉冲计数、累计运行时间、关键事件日志
     *
     * 💡 示例：
     * 如果你每 100ms 记录一次温度，设为 false → 睡醒后补记录所有缺失数据
     * 如果你每 500ms 刷新一次屏幕，设为 true → 睡醒后直接显示最新状态，不补刷
     */
    bool skip_unhandled_events;

} esp_timer_create_args_t;
```

**作用：**传递给esp_timer_create()的定时器配置，用于配置定时器参数

### esp_timer_dispatch_t

**枚举定义：**

```c
typedef enum {
    ESP_TIMER_TASK,     // 从esp_timer任务分发回调
    ESP_TIMER_ISR,      // 从中断处理程序分发回调
    ESP_TIMER_MAX,      // 回调分发方法数量的哨兵值
} esp_timer_dispatch_t;
```

**作用：**分发定时器回调方式的结构体

### esp_timer_cb_t

**函数指针定义：**

```c
typedef void (*esp_timer_cb_t)(void* arg);
```

**作用：**定时器回调函数类型

### esp_timer_handle_t

**类型定义：**

```c
typedef struct esp_timer* esp_timer_handle_t;
```

**作用：**表示单个定时器句柄的不透明类型

## 1. Timer初始化和反初始化函数

### esp_timer_early_init
**函数原型：**
```c
esp_err_t esp_timer_early_init(void);
```
**参数：**
无
**返回值：**
ESP_OK：操作成功
**作用：**Timer的最小初始化，调用此函数后只能使用esp_timer_get_time()函数

### esp_timer_init
**函数原型：**
```c
esp_err_t esp_timer_init(void);
```
**参数：**
无
**返回值：**
ESP_OK：操作成功
ESP_ERR_NO_MEM：内存分配失败
ESP_ERR_INVALID_STATE：已经初始化
其他错误：来自中断分配器的错误
**作用：**初始化esp_timer库，在每个核心上调用

### esp_timer_deinit
**函数原型：**
```c
esp_err_t esp_timer_deinit(void);
```
**参数：**
无
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：尚未初始化
**作用：**去初始化esp_timer库

## 2. Timer创建和删除函数

### esp_timer_create
**函数原型：**
```c
esp_err_t esp_timer_create(const esp_timer_create_args_t* create_args, esp_timer_handle_t* out_handle);
```
**参数：**
create_args：指向包含定时器创建参数的结构体指针
out_handle：输出参数，指向存储创建的定时器句柄的变量指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：某些创建参数无效
ESP_ERR_INVALID_STATE：esp_timer库尚未初始化
ESP_ERR_NO_MEM：内存分配失败
**作用：**创建一个esp_timer实例

### esp_timer_delete
**函数原型：**
```c
esp_err_t esp_timer_delete(esp_timer_handle_t timer);
```
**参数：**
timer：使用esp_timer_create()创建的定时器句柄
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：定时器正在运行
**作用：**删除esp_timer实例，定时器必须先停止

## 3. Timer启动和控制函数

### esp_timer_start_once
**函数原型：**
```c
esp_err_t esp_timer_start_once(esp_timer_handle_t timer, uint64_t timeout_us);
```
**参数：**
timer：使用esp_timer_create()创建的定时器句柄
timeout_us：定时器超时时间，相对于当前时刻的微秒数
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：句柄无效
ESP_ERR_INVALID_STATE：定时器已在运行
**作用：**启动一次性定时器

### esp_timer_start_periodic
**函数原型：**
```c
esp_err_t esp_timer_start_periodic(esp_timer_handle_t timer, uint64_t period);
```
**参数：**
timer：使用esp_timer_create()创建的定时器句柄
period：定时器周期，微秒
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：句柄无效
ESP_ERR_INVALID_STATE：定时器已在运行
**作用：**启动周期性定时器

### esp_timer_restart
**函数原型：**
```c
esp_err_t esp_timer_restart(esp_timer_handle_t timer, uint64_t timeout_us);
```
**参数：**
timer：使用esp_timer_create()创建的定时器句柄
timeout_us：相对于当前时间的超时时间，微秒。对于周期性定时器，也表示新的周期
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：句柄无效
ESP_ERR_INVALID_STATE：定时器未运行
**作用：**重启当前运行的定时器

### esp_timer_stop
**函数原型：**
```c
esp_err_t esp_timer_stop(esp_timer_handle_t timer);
```
**参数：**
timer：使用esp_timer_create()创建的定时器句柄
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：定时器未运行
**作用：**停止运行中的定时器

## 4. Timer状态查询函数

### esp_timer_is_active
**函数原型：**
```c
bool esp_timer_is_active(esp_timer_handle_t timer);
```
**参数：**
timer：使用esp_timer_create()创建的定时器句柄
**返回值：**
1：定时器仍然活跃（运行中）
0：定时器未活跃
**作用：**返回定时器状态，是否活跃

### esp_timer_get_period
**函数原型：**
```c
esp_err_t esp_timer_get_period(esp_timer_handle_t timer, uint64_t *period);
```
**参数：**
timer：使用esp_timer_create()创建的定时器句柄
period：存储定时器周期值的内存，微秒
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数无效
**作用：**获取定时器的周期，一次性定时器的超时周期为0

### esp_timer_get_expiry_time
**函数原型：**
```c
esp_err_t esp_timer_get_expiry_time(esp_timer_handle_t timer, uint64_t *expiry);
```
**参数：**
timer：使用esp_timer_create()创建的定时器句柄
expiry：存储超时值的内存，微秒
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_NOT_SUPPORTED：定时器类型为周期性
**作用：**获取一次性定时器的到期时间

## 5. Timer时间获取函数

### esp_timer_get_time
**函数原型：**
```c
int64_t esp_timer_get_time(void);
```
**参数：**
无
**返回值：**
自ESP Timer初始化以来的微秒数
**作用：**获取自启动以来的微秒时间

### esp_timer_get_next_alarm
**函数原型：**
```c
int64_t esp_timer_get_next_alarm(void);
```
**参数：**
无
**返回值：**
最近定时器事件的时间戳，微秒。时间基准与esp_timer_get_time()返回值相同
**作用：**获取下一个预期超时的时间戳

### esp_timer_get_next_alarm_for_wake_up
**函数原型：**
```c
int64_t esp_timer_get_next_alarm_for_wake_up(void);
```
**参数：**
无
**返回值：**
最近定时器事件的时间戳，微秒。时间基准与esp_timer_get_time()返回值相同
**作用：**获取下一个预期超时的时间戳，排除不应中断轻度睡眠的定时器

## 6. Timer调试和诊断函数

### esp_timer_dump
**函数原型：**
```c
esp_err_t esp_timer_dump(FILE* stream);
```
**参数：**
stream：要输出信息的流（如stdout）
**返回值：**
ESP_OK：操作成功
ESP_ERR_NO_MEM：无法为输出分配临时缓冲区
**作用：**将定时器列表输出到流

## 7. Timer中断相关函数

### esp_timer_isr_dispatch_need_yield
**函数原型：**
```c
void esp_timer_isr_dispatch_need_yield(void);
```
**参数：**
无
**返回值：**
无
**作用：**从定时器回调函数请求上下文切换，仅适用于ISR分发方法的定时器

## 8. Timer ETM事件函数

### esp_timer_new_etm_alarm_event
**函数原型：**
```c
esp_err_t esp_timer_new_etm_alarm_event(esp_etm_event_handle_t *out_event);
```
**参数：**
out_event：返回的ETM事件句柄
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**获取esp_timer底层alarm事件的ETM事件句柄

