# ESP32 LEDC (LED控制器) API 完整函数参考

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
**作用：**配置LEDC通道，包括GPIO引脚、速度模式、通道、中断类型、定时器选择、占空比等

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
ESP_FAIL：无法基于给定频率和占空比分辨率找到合适的预分频器
ESP_ERR_INVALID_STATE：定时器无法被解配置，因为定时器未配置或未暂停
**作用：**配置LEDC定时器，包括速度模式、占空比分辨率、定时器编号、频率和时钟配置

### ledc_find_suitable_duty_resolution
**函数原型：**
```c
uint32_t ledc_find_suitable_duty_resolution(uint32_t src_clk_freq, uint32_t timer_freq);
```
**参数：**
src_clk_freq：LEDC定时器源时钟频率（Hz）
timer_freq：期望的LEDC定时器频率（Hz）
**返回值：**
0：无法实现该定时器频率
其他值：可设置的最大占空比分辨率值
**作用：**帮助函数，用于找到ledc_timer_config()的最大可能占空比分辨率

### ledc_set_pin
**函数原型：**
```c
esp_err_t ledc_set_pin(int gpio_num, ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
gpio_num：LEDC输出GPIO引脚编号
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置LEDC输出GPIO引脚

## 2. LEDC占空比控制函数

### ledc_set_duty
**函数原型：**
```c
esp_err_t ledc_set_duty(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
duty：设置LEDC占空比，范围为[0, (2**duty_resolution)]
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置LEDC占空比，需要调用ledc_update_duty函数后生效

### ledc_get_duty
**函数原型：**
```c
uint32_t ledc_get_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
**返回值：**
LEDC_ERR_DUTY：参数错误
其他值：当前LEDC占空比
**作用：**获取当前PWM周期的占空比

### ledc_set_duty_with_hpoint
**函数原型：**
```c
esp_err_t ledc_set_duty_with_hpoint(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, uint32_t hpoint);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
duty：设置LEDC占空比，范围为[0, (2**duty_resolution)]
hpoint：设置LEDC高点值，范围为[0, (2**duty_resolution)-1]
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置LEDC占空比和高点值，需要调用ledc_update_duty函数后生效

### ledc_get_hpoint
**函数原型：**
```c
int ledc_get_hpoint(ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
**返回值：**
LEDC_ERR_VAL：参数错误
其他值：LEDC通道当前高点值
**作用：**获取高点值，即输出设置为高电平时的计数器值

### ledc_update_duty
**函数原型：**
```c
esp_err_t ledc_update_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**更新LEDC通道参数，使设置的占空比生效

### ledc_set_duty_and_update
**函数原型：**
```c
esp_err_t ledc_set_duty_and_update(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, uint32_t hpoint);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
duty：设置LEDC占空比，范围为[0, (2**duty_resolution)]
hpoint：设置LEDC高点值，范围为[0, (2**duty_resolution)-1]
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：通道未初始化
ESP_ERR_INVALID_ARG：参数错误
ESP_FAIL：淡入淡出函数初始化错误
**作用：**线程安全的API，用于设置LEDC通道占空比并立即更新

## 3. LEDC频率控制函数

### ledc_set_freq
**函数原型：**
```c
esp_err_t ledc_set_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num, uint32_t freq_hz);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
timer_num：LEDC定时器索引（0-3）
freq_hz：设置LEDC频率（Hz）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_FAIL：无法基于给定频率和当前占空比分辨率找到合适的预分频器
**作用：**设置LEDC通道频率

### ledc_get_freq
**函数原型：**
```c
uint32_t ledc_get_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
timer_num：LEDC定时器索引（0-3）
**返回值：**
0：错误
其他值：当前LEDC频率
**作用：**获取LEDC通道频率

## 4. LEDC定时器控制函数

### ledc_timer_set
**函数原型：**
```c
esp_err_t ledc_timer_set(ledc_mode_t speed_mode, ledc_timer_t timer_sel, uint32_t clock_divider, uint32_t duty_resolution, ledc_clk_src_t clk_src);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
timer_sel：定时器索引（0-3）
clock_divider：定时器时钟分频值
duty_resolution：占空比分辨率位数
clk_src：选择LEDC源时钟
**返回值：**
ESP_OK：操作成功
其他值：参数错误
**作用：**配置LEDC定时器设置（已弃用，建议使用ledc_timer_config()或ledc_set_freq()）

### ledc_timer_rst
**函数原型：**
```c
esp_err_t ledc_timer_rst(ledc_mode_t speed_mode, ledc_timer_t timer_sel);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
timer_sel：LEDC定时器索引（0-3）
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
speed_mode：选择指定速度模式的LEDC通道组
timer_sel：LEDC定时器索引（0-3）
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
speed_mode：选择指定速度模式的LEDC通道组
timer_sel：LEDC定时器索引（0-3）
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
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
timer_sel：LEDC定时器索引（0-3）
**返回值：**
ESP_ERR_INVALID_ARG：参数错误
ESP_OK：操作成功
**作用：**将LEDC通道与选定的定时器绑定

## 5. LEDC输出控制函数

### ledc_stop
**函数原型：**
```c
esp_err_t ledc_stop(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t idle_level);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
idle_level：LEDC停止后设置的输出空闲电平
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**停止LEDC输出并设置空闲电平

## 6. LEDC淡入淡出功能函数

### ledc_set_fade_with_time
**函数原型：**
```c
esp_err_t ledc_set_fade_with_time(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, int desired_fade_time_ms);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
target_duty：淡入淡出的目标占空比[0, (2**duty_resolution)]
desired_fade_time_ms：期望的淡入淡出时间（毫秒）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：淡入淡出函数初始化错误
**作用：**设置LEDC淡入淡出函数，在限定时间内完成

### ledc_set_fade_with_step
**函数原型：**
```c
esp_err_t ledc_set_fade_with_step(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, uint32_t scale, uint32_t cycle_num);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
target_duty：淡入淡出的目标占空比[0, (2**duty_resolution)]
scale：控制增加或减少步长比例
cycle_num：每cycle_num个周期增加或减少一次占空比
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：淡入淡出函数初始化错误
**作用：**设置LEDC淡入淡出函数

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
ESP_ERR_NOT_FOUND：找不到可用的中断源
ESP_ERR_INVALID_STATE：淡入淡出函数已安装
**作用：**安装LEDC淡入淡出函数，会占用LEDC模块的中断

### ledc_fade_func_uninstall
**函数原型：**
```c
void ledc_fade_func_uninstall(void);
```
**参数：**
无
**返回值：**
无
**作用：**卸载LEDC淡入淡出函数

### ledc_fade_start
**函数原型：**
```c
esp_err_t ledc_fade_start(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_fade_mode_t fade_mode);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道编号
fade_mode：是否阻塞直到淡入淡出完成
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：通道未初始化或淡入淡出函数未安装
ESP_ERR_INVALID_ARG：参数错误
**作用：**启动LEDC淡入淡出

### ledc_fade_stop
**函数原型：**
```c
esp_err_t ledc_fade_stop(ledc_mode_t speed_mode, ledc_channel_t channel);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_STATE：通道未初始化
ESP_ERR_INVALID_ARG：参数错误
ESP_FAIL：淡入淡出函数初始化错误
**作用：**停止LEDC淡入淡出（仅在某些芯片上支持）

### ledc_set_fade_time_and_start
**函数原型：**
```c
esp_err_t ledc_set_fade_time_and_start(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, uint32_t desired_fade_time_ms, ledc_fade_mode_t fade_mode);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
target_duty：淡入淡出的目标占空比[0, (2**duty_resolution)]
desired_fade_time_ms：期望的淡入淡出时间（毫秒）
fade_mode：选择阻塞或非阻塞模式
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：淡入淡出函数初始化错误
**作用：**线程安全的API，用于设置和启动LEDC淡入淡出函数

### ledc_set_fade_step_and_start
**函数原型：**
```c
esp_err_t ledc_set_fade_step_and_start(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, uint32_t scale, uint32_t cycle_num, ledc_fade_mode_t fade_mode);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
target_duty：淡入淡出的目标占空比[0, (2**duty_resolution)]
scale：控制增加或减少步长比例
cycle_num：每cycle_num个周期增加或减少一次占空比
fade_mode：选择阻塞或非阻塞模式
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：淡入淡出函数初始化错误
**作用：**线程安全的API，用于设置和启动LEDC淡入淡出函数

## 7. LEDC多段淡入淡出函数（高级功能）

### ledc_set_multi_fade
**函数原型：**
```c
esp_err_t ledc_set_multi_fade(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t start_duty, const ledc_fade_param_config_t *fade_params_list, uint32_t list_len);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
start_duty：设置渐变占空比的起始值[0, (2**duty_resolution)]
fade_params_list：指向多段淡入淡出参数数组的指针
list_len：fade_params_list的长度，即多段淡入淡出的范围数量
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：淡入淡出函数初始化错误
**作用：**设置LEDC多段淡入淡出（仅在支持伽马曲线淡入淡出的芯片上可用）

### ledc_set_multi_fade_and_start
**函数原型：**
```c
esp_err_t ledc_set_multi_fade_and_start(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t start_duty, const ledc_fade_param_config_t *fade_params_list, uint32_t list_len, ledc_fade_mode_t fade_mode);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
start_duty：设置渐变占空比的起始值[0, (2**duty_resolution)]
fade_params_list：指向多段淡入淡出参数数组的指针
list_len：fade_params_list的长度
fade_mode：选择阻塞或非阻塞模式
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：淡入淡出函数初始化错误
**作用：**线程安全的API，用于设置和启动LEDC多段淡入淡出函数

### ledc_fill_multi_fade_param_list
**函数原型：**
```c
esp_err_t ledc_fill_multi_fade_param_list(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t start_duty, uint32_t end_duty, uint32_t linear_phase_num, uint32_t max_fade_time_ms, uint32_t (* gamma_correction_operator)(uint32_t), uint32_t fade_params_list_size, ledc_fade_param_config_t *fade_params_list, uint32_t *hw_fade_range_num);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
start_duty：多段淡入淡出开始的占空比
end_duty：多段淡入淡出结束的占空比
linear_phase_num：模拟伽马曲线淡入淡出的线性淡入淡出数量
max_fade_time_ms：淡入淡出的最大时间（毫秒）
gamma_correction_operator：用户提供的伽马校正函数
fade_params_list_size：用户分配的fade_params_list的大小
fade_params_list：指向ledc_fade_param_config_t结构体数组的指针
hw_fade_range_num：此多段淡入淡出的淡入淡出范围数量
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：所需的硬件范围数量超过用户分配的数组大小
**作用：**帮助函数，用于填充多段淡入淡出的淡入淡出参数

### ledc_read_fade_param
**函数原型：**
```c
esp_err_t ledc_read_fade_param(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t range, uint32_t *dir, uint32_t *cycle, uint32_t *scale, uint32_t *step);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
range：范围索引，指定从gamma RAM的哪个范围读取
dir：指向接受淡入淡出方向值的指针
cycle：指向接受淡入淡出周期值的指针
scale：指向接受淡入淡出比例值的指针
step：指向接受淡入淡出步进值的指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
**作用：**获取gamma RAM中某个淡入淡出范围存储的淡入淡出参数

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
handle：指向返回句柄的指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_NOT_FOUND：找不到可用的中断源
**作用：**注册LEDC中断处理程序

### ledc_cb_register
**函数原型：**
```c
esp_err_t ledc_cb_register(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_cbs_t *cbs, void *user_arg);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道索引（0 - LEDC_CHANNEL_MAX-1）
cbs：LEDC回调函数组
user_arg：回调函数的用户注册数据
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：通道未初始化
ESP_FAIL：淡入淡出函数初始化错误
**作用：**LEDC回调注册函数

## 9. LEDC渐变设置函数（低级API）

### ledc_set_fade
**函数原型：**
```c
esp_err_t ledc_set_fade(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, ledc_duty_direction_t fade_direction, uint32_t step_num, uint32_t duty_cycle_num, uint32_t duty_scale);
```
**参数：**
speed_mode：选择指定速度模式的LEDC通道组
channel：LEDC通道（0 - LEDC_CHANNEL_MAX-1）
duty：设置渐变占空比的起始值
fade_direction：设置渐变的方向
step_num：设置渐变的步数
duty_cycle_num：设置每个渐变持续多少个LEDC tick
duty_scale：设置渐变变化幅度
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置LEDC渐变，调用ledc_update_duty函数后生效
