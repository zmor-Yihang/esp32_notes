# ESP32 LEDC API 完整函数参考

## 0. LEDC主要数据结构

### ledc_channel_config_t

LEDC通道配置结构体，用于配置LEDC通道的所有参数：

```c
typedef struct {
    int gpio_num;                      // GPIO编号
    ledc_mode_t speed_mode;           // 速度模式
    ledc_channel_t channel;           // LEDC通道
    ledc_intr_type_t intr_type;       // 中断类型
    ledc_timer_t timer_sel;           // 定时器选择
    uint32_t duty;                    // 占空比
    int hpoint;                       // H点值
    ledc_sleep_mode_t sleep_mode;     // 睡眠模式
    struct {
        unsigned int output_invert: 1; // 输出反转标志
    } flags;
} ledc_channel_config_t;
```

**详细成员说明：**

- **`gpio_num`** (类型: `int`)
  - **功能**: 指定LEDC输出的GPIO引脚编号
  - **取值范围**: 有效的GPIO引脚编号 (0-39，具体取决于芯片型号)
  - **注意事项**: 必须是可输出的GPIO引脚，某些引脚可能有特殊用途
  - **示例**: `GPIO_NUM_2`, `GPIO_NUM_5`

- **`speed_mode`** (类型: `ledc_mode_t`)
  - **功能**: 选择LEDC速度模式，影响时钟源和功耗
  - **可选值**: 
    - `LEDC_LOW_SPEED_MODE`: 低速模式，支持更多功能但功耗稍高
    - `LEDC_HIGH_SPEED_MODE`: 高速模式 (某些芯片不支持)
  - **使用建议**: ESP32-S2/S3/C3等新芯片建议使用低速模式
  - **注意事项**: 高速模式和低速模式的定时器和通道是独立的

- **`channel`** (类型: `ledc_channel_t`)
  - **功能**: 指定使用的LEDC通道编号
  - **取值范围**: `LEDC_CHANNEL_0` 到 `LEDC_CHANNEL_7` (0-7)
  - **说明**: 每种速度模式下有8个独立的通道可用
  - **注意事项**: 不同通道可以使用相同的定时器，但会有相同的频率

- **`intr_type`** (类型: `ledc_intr_type_t`)
  - **功能**: 配置中断类型，主要用于渐变效果
  - **可选值**:
    - `LEDC_INTR_DISABLE`: 禁用中断
    - `LEDC_INTR_FADE_END`: 渐变结束中断
  - **使用场景**: 当需要在渐变完成时执行回调函数时使用
  - **注意事项**: 需要配合中断服务程序使用

- **`timer_sel`** (类型: `ledc_timer_t`)
  - **功能**: 选择该通道使用的定时器
  - **取值范围**: `LEDC_TIMER_0` 到 `LEDC_TIMER_3` (0-3)
  - **说明**: 每种速度模式下有4个定时器可用
  - **共享机制**: 多个通道可以共享同一个定时器，共享频率和分辨率设置

- **`duty`** (类型: `uint32_t`)
  - **功能**: 设置PWM信号的占空比值
  - **取值范围**: 0 到 `(2^duty_resolution - 1)`
  - **计算公式**: 实际占空比 = duty / (2^duty_resolution)
  - **示例**: 如果分辨率为8位，duty=128时占空比为50%
  - **注意事项**: duty值不能超过当前分辨率的最大值

- **`hpoint`** (类型: `int`)
  - **功能**: 设置PWM波形的起始高电平点
  - **取值范围**: 0 到 `(2^duty_resolution - 1)`
  - **作用**: 控制PWM波形在一个周期内的相位偏移
  - **默认值**: 通常设置为0
  - **应用场景**: 多通道相位错开，减少同时开关带来的电流冲击

- **`sleep_mode`** (类型: `ledc_sleep_mode_t`)
  - **功能**: 定义系统进入Light Sleep模式时通道的行为**(已弃用)**
  - **可选值**:
    - `LEDC_SLEEP_MODE_CONTINUOUS`: 睡眠时继续运行
    - `LEDC_SLEEP_MODE_PAUSE`: 睡眠时暂停输出
  - **功耗考虑**: PAUSE模式可以节省更多功耗
  - **应用场景**: 根据应用需求选择是否需要在睡眠时保持PWM输出

- **`flags.output_invert`** (类型: `unsigned int`, 1位)
  - **功能**: 控制GPIO输出信号的极性
  - **取值**: 0-正常输出，1-反转输出
  - **应用**: 当需要反转PWM信号时使用（高电平变低电平，低电平变高电平）
  - **硬件实现**: 在GPIO驱动级别实现反转

### ledc_timer_config_t

LEDC定时器配置结构体，用于配置PWM信号的频率和分辨率：

```c
typedef struct {
    ledc_mode_t speed_mode;           // 速度模式
    ledc_timer_bit_t duty_resolution; // 占空比分辨率
    ledc_timer_t timer_num;           // 定时器编号
    uint32_t freq_hz;                 // 频率
    ledc_clk_cfg_t clk_cfg;          // 时钟配置
    bool deconfigure;                 // 取消配置标志
} ledc_timer_config_t;
```

**详细成员说明：**

- **`speed_mode`** (类型: `ledc_mode_t`)
  - **功能**: 与通道配置中的speed_mode对应
  - **作用**: 决定定时器属于高速模式还是低速模式
  - **约束**: 必须与使用该定时器的通道的速度模式匹配

- **`duty_resolution`** (类型: `ledc_timer_bit_t`)
  - **功能**: 设置PWM占空比的分辨率位数
  - **可选值**: `LEDC_TIMER_1_BIT` 到 `LEDC_TIMER_20_BIT` (1-20位)
  - **影响**: 分辨率越高，占空比调节越精细，但可达到的最高频率越低
  - **计算关系**: 最大占空比值 = 2^duty_resolution - 1
  - **常用值**: 8位(256级)、10位(1024级)、13位(8192级)
  - **频率限制**: 高分辨率会限制可达到的最高频率

- **`timer_num`** (类型: `ledc_timer_t`)
  - **功能**: 指定配置哪个定时器
  - **取值范围**: `LEDC_TIMER_0` 到 `LEDC_TIMER_3`
  - **独立性**: 每个速度模式下的4个定时器完全独立
  - **复用**: 一个定时器可以被多个通道共享使用

- **`freq_hz`** (类型: `uint32_t`)
  - **功能**: 设置PWM信号的基本频率（单位：Hz）
  - **取值范围**: 1Hz 到几MHz（具体上限取决于时钟源和分辨率）
  - **计算限制**: freq_hz ≤ 时钟源频率 / (2^duty_resolution)
  - **精度**: 实际频率可能与设定值有微小差异
  - **应用指导**:
    - LED调光: 1kHz-5kHz（避免可见闪烁）
    - 电机控制: 20kHz-50kHz（超出人耳听觉范围）
    - 舵机控制: 50Hz（标准舵机频率）

- **`clk_cfg`** (类型: `ledc_clk_cfg_t`)
  - **功能**: 配置定时器使用的时钟源
  - **可选值**:
    - `LEDC_AUTO_CLK`: 自动选择合适的时钟源（推荐）
    - `LEDC_USE_APB_CLK`: 使用APB时钟（80MHz）
    - `LEDC_USE_RTC8M_CLK`: 使用RTC 8MHz时钟（低功耗）
    - `LEDC_USE_XTAL_CLK`: 使用外部晶振时钟
  - **选择原则**: AUTO模式会根据频率需求自动选择最合适的时钟源
  - **功耗考虑**: RTC时钟功耗最低，但频率范围有限

- **`deconfigure`** (类型: `bool`)
  - **功能**: 控制是否取消之前的定时器配置
  - **取值**: `true`-取消配置，`false`-正常配置
  - **使用场景**: 当需要完全停止某个定时器时设置为true
  - **注意事项**: 取消配置会影响所有使用该定时器的通道

### ledc_fade_param_config_t (支持伽马曲线的芯片)

单个硬件渐变的渐变参数结构体，用于精确控制PWM渐变效果：

```c
typedef struct {
    uint32_t dir: 1;                  // 渐变方向
    uint32_t cycle_num;               // 每步周期数
    uint32_t scale;                   // 每步变化量
    uint32_t step_num;                // 总步数
} ledc_fade_param_config_t;
```

**详细成员说明：**

- **`dir`** (类型: `uint32_t`, 1位)
  - **功能**: 控制占空比变化的方向
  - **取值**: 1-递增（占空比增加），0-递减（占空比减少）
  - **应用**: 配合scale和step_num实现线性或非线性渐变

- **`cycle_num`** (类型: `uint32_t`)
  - **功能**: 指定每个渐变步骤持续的PWM周期数
  - **取值范围**: 0 到 `(2^SOC_LEDC_FADE_PARAMS_BIT_WIDTH - 1)`
  - **影响**: 值越大，渐变速度越慢，渐变越平滑
  - **计算**: 每步持续时间 = cycle_num × PWM周期时间

- **`scale`** (类型: `uint32_t`)
  - **功能**: 定义每个渐变步骤的占空比变化量
  - **取值范围**: 0 到 `(2^SOC_LEDC_FADE_PARAMS_BIT_WIDTH - 1)`
  - **非线性**: 可以为不同步骤设置不同的scale值实现非线性渐变
  - **精度**: scale值决定了渐变的细腻程度

- **`step_num`** (类型: `uint32_t`)
  - **功能**: 指定整个渐变过程包含的步骤总数
  - **取值范围**: 0 到 `(2^SOC_LEDC_FADE_PARAMS_BIT_WIDTH - 1)`
  - **计算**: 总渐变时间 = step_num × cycle_num × PWM周期时间
  - **应用**: 与scale配合可以实现复杂的渐变曲线

**渐变参数配置示例：**

```c
// 创建一个从0%到100%的线性渐变，持续2秒
ledc_fade_param_config_t fade_params[] = {
    {.dir = 1, .cycle_num = 1000, .scale = 1, .step_num = 255}
};
```

**伽马校正应用：**
- 支持伽马曲线的芯片可以使用此结构体数组定义复杂的亮度曲线
- 通过调整不同步骤的scale值，可以实现人眼感知线性的亮度变化
- 常用于LED亮度控制，实现更自然的渐变效果

## 1. LEDC初始化和配置函数

### ledc_channel_config
**函数原型：**
```c
esp_err_t ledc_channel_config(const ledc_channel_config_t *ledc_conf);
```
**参数：**
ledc_conf：指向包含LEDC通道配置信息的ledc_channel_config_t结构体的指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**根据ledc_channel_config_t中指定的参数配置LEDC通道，包括GPIO引脚、速度模式、通道、中断类型、定时器选择、占空比等

### ledc_timer_config
**函数原型：**
```c
esp_err_t ledc_timer_config(const ledc_timer_config_t *timer_conf);
```
**参数：**
timer_conf：指向包含LEDC定时器配置信息的ledc_timer_config_t结构体的指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_FAIL：无法找到合适的预分频数
ESP_ERR_INVALID_STATE：定时器无法取消配置
**作用：**配置LEDC定时器，设置源定时器、频率、占空比分辨率

### ledc_find_suitable_duty_resolution
**函数原型：**
```c
uint32_t ledc_find_suitable_duty_resolution(uint32_t src_clk_freq, uint32_t timer_freq);
```
**参数：**
src_clk_freq：LEDC定时器源时钟频率(Hz)
timer_freq：期望的LEDC定时器频率(Hz)
**返回值：**
0：无法达到该定时器频率
其他值：可设置的最大占空比分辨率值
**作用：**帮助函数，用于为ledc_timer_config()找到最大可能的占空比分辨率

## 2. LEDC输出控制函数

### ledc_update_duty
**函数原型：**
```c
esp_err_t ledc_update_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**更新LEDC通道参数，在ledc_set_duty后需要调用此函数来应用新设置

### ledc_set_pin
**函数原型：**
```c
esp_err_t ledc_set_pin(int gpio_num, ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
gpio_num：LEDC输出GPIO引脚编号
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置LEDC输出GPIO，仅通过矩阵将LEDC信号路由到GPIO

### ledc_stop
**函数原型：**
```c
esp_err_t ledc_stop(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t idle_level);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
idle_level：LEDC停止后设置的输出空闲电平
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**停止LEDC输出，禁用LEDC输出并设置空闲电平

## 3. LEDC频率控制函数

### ledc_set_freq
**函数原型：**
```c
esp_err_t ledc_set_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num, uint32_t freq_hz);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
timer_num：LEDC定时器索引(0-3)
freq_hz：设置的LEDC频率
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_FAIL：无法找到合适的预分频数
**作用：**设置LEDC通道频率(Hz)

### ledc_get_freq
**函数原型：**
```c
uint32_t ledc_get_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
timer_num：LEDC定时器索引(0-3)
**返回值：**
0：错误
其他值：当前LEDC频率
**作用：**获取LEDC通道频率(Hz)

## 4. LEDC占空比控制函数

### ledc_set_duty
**函数原型：**
```c
esp_err_t ledc_set_duty(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
duty：设置的LEDC占空比，范围[0, (2**duty_resolution)]
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置LEDC占空比，不改变此通道的hpoint值，需要调用ledc_update_duty使占空比更新生效

### ledc_get_duty
**函数原型：**
```c
uint32_t ledc_get_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
**返回值：**
LEDC_ERR_DUTY：参数错误
其他值：当前LEDC占空比
**作用：**获取LEDC占空比，返回当前PWM周期的占空比

### ledc_set_duty_with_hpoint
**函数原型：**
```c
esp_err_t ledc_set_duty_with_hpoint(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, uint32_t hpoint);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
duty：设置的LEDC占空比，范围[0, (2**duty_resolution)]
hpoint：设置的LEDC hpoint值，范围[0, (2**duty_resolution)-1]
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置LEDC占空比和hpoint值，只有调用ledc_update_duty后占空比才会更新

### ledc_get_hpoint
**函数原型：**
```c
int ledc_get_hpoint(ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
**返回值：**
LEDC_ERR_VAL：参数错误
其他值：LEDC通道当前hpoint值
**作用：**获取LEDC hpoint值，即输出设置为高电平时的计数器值

### ledc_set_duty_and_update
**函数原型：**
```c
esp_err_t ledc_set_duty_and_update(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, uint32_t hpoint);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
duty：设置的LEDC占空比，范围[0, (2**duty_resolution)]
hpoint：设置的LEDC hpoint值，范围[0, (2**duty_resolution)-1]
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：通道未初始化
ESP_ERR_INVALID_ARG：参数错误
ESP_FAIL：渐变功能初始化错误
**作用：**线程安全的API，设置LEDC通道占空比并在占空比更新后返回

## 5. LEDC定时器控制函数

### ledc_timer_set (已弃用)
**函数原型：**
```c
esp_err_t ledc_timer_set(ledc_mode_t speed_mode, ledc_timer_t timer_sel, uint32_t clock_divider, uint32_t duty_resolution, ledc_clk_src_t clk_src);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
timer_sel：定时器索引(0-3)
clock_divider：定时器时钟分频值
duty_resolution：占空比设置的分辨率位数
clk_src：选择LEDC源时钟
**返回值：**
ESP_ERR_INVALID_ARG：参数错误
ESP_OK：操作成功
**作用：**配置LEDC定时器设置（已弃用，请使用ledc_timer_config()或ledc_set_freq()）

### ledc_timer_rst
**函数原型：**
```c
esp_err_t ledc_timer_rst(ledc_mode_t speed_mode, ledc_timer_t timer_sel);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
timer_sel：LEDC定时器索引(0-3)
**返回值：**
ESP_ERR_INVALID_ARG：参数错误
ESP_OK：操作成功
**作用：**重置LEDC定时器

### ledc_timer_pause
**函数原型：**
```c
esp_err_t ledc_timer_pause(ledc_mode_t speed_mode, ledc_timer_t timer_sel);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
timer_sel：LEDC定时器索引(0-3)
**返回值：**
ESP_ERR_INVALID_ARG：参数错误
ESP_OK：操作成功
**作用：**暂停LEDC定时器计数器

### ledc_timer_resume
**函数原型：**
```c
esp_err_t ledc_timer_resume(ledc_mode_t speed_mode, ledc_timer_t timer_sel);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
timer_sel：LEDC定时器索引(0-3)
**返回值：**
ESP_ERR_INVALID_ARG：参数错误
ESP_OK：操作成功
**作用：**恢复LEDC定时器

### ledc_bind_channel_timer
**函数原型：**
```c
esp_err_t ledc_bind_channel_timer(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_timer_t timer_sel);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
timer_sel：LEDC定时器索引(0-3)
**返回值：**
ESP_ERR_INVALID_ARG：参数错误
ESP_OK：操作成功
**作用：**将LEDC通道与选定的定时器绑定

## 6. LEDC渐变功能函数

### ledc_fade_func_install
**函数原型：**
```c
esp_err_t ledc_fade_func_install(int intr_alloc_flags);
```
**参数：**
intr_alloc_flags：用于分配中断的标志
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：中断标志错误
ESP_ERR_NOT_FOUND：未找到可用的中断源
ESP_ERR_INVALID_STATE：渐变功能已安装
**作用：**安装LEDC渐变功能，此函数将占用LEDC模块的中断

### ledc_fade_func_uninstall
**函数原型：**
```c
void ledc_fade_func_uninstall(void);
```
**参数：**
无
**返回值：**
无
**作用：**卸载LEDC渐变功能

### ledc_set_fade_with_step
**函数原型：**
```c
esp_err_t ledc_set_fade_with_step(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, uint32_t scale, uint32_t cycle_num);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
target_duty：渐变的目标占空比[0, (2**duty_resolution)]
scale：控制增加或减少步长的尺度
cycle_num：每cycle_num个周期增加或减少占空比
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：渐变功能初始化错误
**作用：**设置LEDC渐变功能，调用ledc_fade_start()开始渐变

### ledc_set_fade_with_time
**函数原型：**
```c
esp_err_t ledc_set_fade_with_time(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, int desired_fade_time_ms);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
target_duty：渐变的目标占空比[0, (2**duty_resolution)]
desired_fade_time_ms：渐变的预期时间(ms)
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：渐变功能初始化错误
**作用：**设置LEDC渐变功能，在限定时间内完成渐变

### ledc_fade_start
**函数原型：**
```c
esp_err_t ledc_fade_start(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_fade_mode_t fade_mode);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道编号
fade_mode：是否阻塞直到渐变完成
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：通道未初始化或渐变功能未安装
ESP_ERR_INVALID_ARG：参数错误
**作用：**启动LEDC渐变，在ledc_set_fade_with_time或ledc_set_fade_with_step之后调用此API开始渐变

### ledc_fade_stop
**函数原型：**
```c
esp_err_t ledc_fade_stop(ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：通道未初始化
ESP_ERR_INVALID_ARG：参数错误
ESP_FAIL：渐变功能初始化错误
**作用：**停止LEDC渐变，保证通道的占空比在函数返回后最多一个PWM周期内固定

### ledc_set_fade_time_and_start
**函数原型：**
```c
esp_err_t ledc_set_fade_time_and_start(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, uint32_t desired_fade_time_ms, ledc_fade_mode_t fade_mode);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
target_duty：渐变的目标占空比[0, (2**duty_resolution)]
desired_fade_time_ms：渐变的预期时间(ms)
fade_mode：选择阻塞或非阻塞模式
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：渐变功能初始化错误
**作用：**线程安全的API，设置并启动LEDC渐变功能，在限定时间内完成渐变

### ledc_set_fade_step_and_start
**函数原型：**
```c
esp_err_t ledc_set_fade_step_and_start(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, uint32_t scale, uint32_t cycle_num, ledc_fade_mode_t fade_mode);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
target_duty：渐变的目标占空比[0, (2**duty_resolution)]
scale：控制增加或减少步长的尺度
cycle_num：每cycle_num个周期增加或减少占空比
fade_mode：选择阻塞或非阻塞模式
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：渐变功能初始化错误
**作用：**线程安全的API，设置并启动LEDC渐变功能

## 7. LEDC渐变设置函数

### ledc_set_fade
**函数原型：**
```c
esp_err_t ledc_set_fade(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, ledc_duty_direction_t fade_direction, uint32_t step_num, uint32_t duty_cycle_num, uint32_t duty_scale);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道(0 - LEDC_CHANNEL_MAX-1)
duty：设置渐变占空比的起始值，范围[0, (2**duty_resolution)]
fade_direction：设置渐变的方向
step_num：设置渐变的步数
duty_cycle_num：设置每次渐变持续多少个LEDC tick
duty_scale：设置渐变变化幅度
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置LEDC渐变，调用ledc_update_duty函数后，该函数才能生效

## 8. LEDC中断和回调函数

### ledc_isr_register
**函数原型：**
```c
esp_err_t ledc_isr_register(void (*fn)(void *), void *arg, int intr_alloc_flags, ledc_isr_handle_t *handle);
```
**参数：**
fn：中断处理函数
arg：传递给处理函数的用户提供参数
intr_alloc_flags：用于分配中断的标志
handle：返回处理句柄的指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_NOT_FOUND：未找到可用的中断源
**作用：**注册LEDC中断处理程序，处理程序是ISR

### ledc_cb_register
**函数原型：**
```c
esp_err_t ledc_cb_register(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_cbs_t *cbs, void *user_arg);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
cbs：LEDC回调函数组
user_arg：为回调函数注册的用户数据
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：渐变功能初始化错误
**作用：**LEDC回调注册函数，回调在ISR环境下运行

## 9. LEDC多级渐变功能函数 (支持伽马曲线的芯片)

### ledc_set_multi_fade
**函数原型：**
```c
esp_err_t ledc_set_multi_fade(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t start_duty, const ledc_fade_param_config_t *fade_params_list, uint32_t list_len);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
start_duty：设置渐变占空比的起始值，范围[0, (2**duty_resolution)]
fade_params_list：指向多级渐变的渐变参数数组的指针
list_len：fade_params_list的长度，即多级渐变的渐变范围数(1 - SOC_LEDC_GAMMA_CURVE_FADE_RANGE_MAX)
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：渐变功能初始化错误
**作用：**设置LEDC多级渐变，调用ledc_fade_start()开始渐变

### ledc_set_multi_fade_and_start
**函数原型：**
```c
esp_err_t ledc_set_multi_fade_and_start(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t start_duty, const ledc_fade_param_config_t *fade_params_list, uint32_t list_len, ledc_fade_mode_t fade_mode);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
start_duty：设置渐变占空比的起始值，范围[0, (2**duty_resolution)]
fade_params_list：指向多级渐变的渐变参数数组的指针
list_len：fade_params_list的长度，即多级渐变的渐变范围数(1 - SOC_LEDC_GAMMA_CURVE_FADE_RANGE_MAX)
fade_mode：选择阻塞或非阻塞模式
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：渐变功能初始化错误
**作用：**线程安全的API，设置并启动LEDC多级渐变功能

### ledc_fill_multi_fade_param_list
**函数原型：**
```c
esp_err_t ledc_fill_multi_fade_param_list(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t start_duty, uint32_t end_duty, uint32_t linear_phase_num, uint32_t max_fade_time_ms, uint32_t (*gamma_correction_operator)(uint32_t), uint32_t fade_params_list_size, ledc_fade_param_config_t *fade_params_list, uint32_t *hw_fade_range_num);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
start_duty：多级渐变开始的占空比[0, (2**duty_resolution)]，应为非伽马校正的占空比
end_duty：多级渐变结束的占空比[0, (2**duty_resolution)]，应为非伽马校正的占空比
linear_phase_num：模拟伽马曲线渐变的线性渐变数量(1 - SOC_LEDC_GAMMA_CURVE_FADE_RANGE_MAX)
max_fade_time_ms：渐变的最大时间(ms)
gamma_correction_operator：用户提供的伽马校正函数
fade_params_list_size：用户分配的fade_params_list的大小(1 - SOC_LEDC_GAMMA_CURVE_FADE_RANGE_MAX)
fade_params_list：指向ledc_fade_param_config_t结构体数组的指针
hw_fade_range_num：此多级渐变的渐变范围数量
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：所需的硬件范围数量超过用户分配的ledc_fade_param_config_t数组大小
**作用：**帮助函数，填充多级渐变的渐变参数，用于伽马曲线渐变

### ledc_read_fade_param
**函数原型：**
```c
esp_err_t ledc_read_fade_param(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t range, uint32_t *dir, uint32_t *cycle, uint32_t *scale, uint32_t *step);
```
**参数：**
speed_mode：选择LEDC通道组的速度模式
channel：LEDC通道索引(0 - LEDC_CHANNEL_MAX-1)
range：范围索引(0 - (SOC_LEDC_GAMMA_CURVE_FADE_RANGE_MAX-1))，指定要从伽马RAM的哪个范围读取
dir：接受渐变方向值的指针
cycle：接受渐变周期值的指针
scale：接受渐变尺度值的指针
step：接受渐变步数值的指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
**作用：**获取存储在伽马RAM中某个渐变范围的渐变参数