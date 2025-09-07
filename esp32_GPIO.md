# ESP32 GPIO API 完整函数参考

## 1. GPIO初始化和配置函数

### gpio_config
**函数原型：**
```c
esp_err_t gpio_config(const gpio_config_t *pGPIOConfig);
```
**参数：**
pGPIOConfig：指向包含GPIO配置信息的gpio_config_t结构体的指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_FAIL：操作失败
**作用：**根据gpio_config_t中指定的参数配置GPIO

### gpio_reset_pin
**函数原型：**
```c
esp_err_t gpio_reset_pin(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号（0 ~ GPIO_PIN_COUNT-1）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**将GPIO重置为默认状态

### gpio_set_direction
**函数原型：**
```c
esp_err_t gpio_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
```
**参数：**
gpio_num：GPIO引脚编号
mode：GPIO模式（GPIO_MODE_INPUT/GPIO_MODE_OUTPUT等）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置某个 GPIO 引脚的工作模式（输入、输出、开漏）

### gpio_input_enable
**函数原型：**
```c
esp_err_t gpio_input_enable(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**使能GPIO输入功能

## 2. GPIO电平控制函数

### gpio_set_level
**函数原型：**
```c
esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level);
```
**参数：**
gpio_num：GPIO引脚编号
level：输出电平（0：低电平，1：高电平）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置GPIO输出电平

### gpio_get_level
**函数原型：**
```c
int gpio_get_level(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
0：低电平
1：高电平
**作用：**获取GPIO输入电平

## 3. GPIO上下拉控制函数

### gpio_set_pull_mode
**函数原型：**
```c
esp_err_t gpio_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);
```
**参数：**
gpio_num：GPIO引脚编号
pull：上下拉模式（GPIO_PULLUP_ONLY/GPIO_PULLDOWN_ONLY等）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置GPIO上下拉模式

### gpio_pullup_en
**函数原型：**
```c
esp_err_t gpio_pullup_en(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**使能GPIO上拉电阻

### gpio_pullup_dis
**函数原型：**
```c
esp_err_t gpio_pullup_dis(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**禁用GPIO上拉电阻

### gpio_pulldown_en
**函数原型：**
```c
esp_err_t gpio_pulldown_en(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**使能GPIO下拉电阻

### gpio_pulldown_dis
**函数原型：**
```c
esp_err_t gpio_pulldown_dis(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**禁用GPIO下拉电阻

## 4. GPIO驱动能力控制函数

### gpio_set_drive_capability
**函数原型：**
```c
esp_err_t gpio_set_drive_capability(gpio_num_t gpio_num, gpio_drive_cap_t strength);
```
**参数：**
gpio_num：GPIO引脚编号
strength：驱动能力（GPIO_DRIVE_CAP_0 ~ GPIO_DRIVE_CAP_3）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置GPIO驱动能力

### gpio_get_drive_capability
**函数原型：**
```c
esp_err_t gpio_get_drive_capability(gpio_num_t gpio_num, gpio_drive_cap_t *strength);
```
**参数：**
gpio_num：GPIO引脚编号
strength：返回的驱动能力值
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**获取GPIO当前驱动能力

## 5. GPIO中断控制函数

### gpio_set_intr_type
**函数原型：**
```c
esp_err_t gpio_set_intr_type(gpio_num_t gpio_num, gpio_int_type_t intr_type);
```
**参数：**
gpio_num：GPIO引脚编号
intr_type：中断类型（GPIO_INTR_POSEDGE/GPIO_INTR_NEGEDGE等）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置GPIO中断触发类型

### gpio_intr_enable
**函数原型：**
```c
esp_err_t gpio_intr_enable(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**使能GPIO中断

### gpio_intr_disable
**函数原型：**
```c
esp_err_t gpio_intr_disable(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**禁用GPIO中断

### gpio_isr_register
**函数原型：**
```c
esp_err_t gpio_isr_register(void (*fn)(void *), void *arg, int intr_alloc_flags, gpio_isr_handle_t *handle);
```
**参数：**
fn：中断处理函数指针
arg：传递给中断处理函数的参数
intr_alloc_flags：中断分配标志
handle：返回的中断句柄
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_NO_MEM：内存不足
ESP_ERR_NOT_FOUND：未找到指定标志的空闲中断
**作用：**注册GPIO中断服务

### gpio_install_isr_service
**函数原型：**
```c
esp_err_t gpio_install_isr_service(int intr_alloc_flags);
```
**参数：**
intr_alloc_flags：中断分配标志
**返回值：**
ESP_OK：操作成功
ESP_ERR_NO_MEM：内存不足
ESP_ERR_INVALID_STATE：服务已安装
ESP_ERR_NOT_FOUND：未找到指定标志的空闲中断
ESP_ERR_INVALID_ARG：参数错误
**作用：**安装GPIO中断服务

### gpio_uninstall_isr_service
**函数原型：**
```c
void gpio_uninstall_isr_service(void);
```
**参数：**
无
**返回值：**
无
**作用：**卸载GPIO中断服务

### gpio_isr_handler_add
**函数原型：**
```c
esp_err_t gpio_isr_handler_add(gpio_num_t gpio_num, gpio_isr_t isr_handler, void *args);
```
**参数：**
gpio_num：GPIO引脚编号
isr_handler：中断处理函数指针
args：传递给中断处理函数的参数
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：服务未安装
**作用：**为指定GPIO引脚添加中断处理函数

### gpio_isr_handler_remove
**函数原型：**
```c
esp_err_t gpio_isr_handler_remove(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
ESP_ERR_INVALID_STATE：服务未安装
**作用：**移除指定GPIO引脚的中断处理函数

## 6. GPIO电源管理函数

### gpio_wakeup_enable
**函数原型：**
```c
esp_err_t gpio_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type);
```
**参数：**
gpio_num：GPIO引脚编号
intr_type：唤醒触发类型（仅支持GPIO_INTR_LOW_LEVEL或GPIO_INTR_HIGH_LEVEL）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**使能GPIO唤醒功能

### gpio_wakeup_disable
**函数原型：**
```c
esp_err_t gpio_wakeup_disable(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**禁用GPIO唤醒功能

### gpio_deep_sleep_wakeup_enable
**函数原型：**
```c
esp_err_t gpio_deep_sleep_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type);
```
**参数：**
gpio_num：GPIO引脚编号
intr_type：唤醒触发类型（仅支持GPIO_INTR_LOW_LEVEL或GPIO_INTR_HIGH_LEVEL）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**使能GPIO深度睡眠唤醒功能

### gpio_deep_sleep_wakeup_disable
**函数原型：**
```c
esp_err_t gpio_deep_sleep_wakeup_disable(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**禁用GPIO深度睡眠唤醒功能

## 7. GPIO保持功能函数

### gpio_hold_en
**函数原型：**
```c
esp_err_t gpio_hold_en(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_NOT_SUPPORTED：不支持保持功能
**作用：**使能GPIO保持功能

### gpio_hold_dis
**函数原型：**
```c
esp_err_t gpio_hold_dis(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_NOT_SUPPORTED：不支持保持功能
**作用：**禁用GPIO保持功能

### gpio_deep_sleep_hold_en
**函数原型：**
```c
void gpio_deep_sleep_hold_en(void);
```
**参数：**
无
**返回值：**
无
**作用：**使能深度睡眠期间的GPIO保持功能

### gpio_deep_sleep_hold_dis
**函数原型：**
```c
void gpio_deep_sleep_hold_dis(void);
```
**参数：**
无
**返回值：**
无
**作用：**禁用深度睡眠期间的GPIO保持功能

### gpio_force_hold_all
**函数原型：**
```c
esp_err_t gpio_force_hold_all(void);
```
**参数：**
无
**返回值：**
ESP_OK：操作成功
**作用：**强制保持所有数字和RTC GPIO

### gpio_force_unhold_all
**函数原型：**
```c
esp_err_t gpio_force_unhold_all(void);
```
**参数：**
无
**返回值：**
ESP_OK：操作成功
**作用：**取消强制保持所有数字和RTC GPIO

## 8. GPIO睡眠控制函数

### gpio_sleep_sel_en
**函数原型：**
```c
esp_err_t gpio_sleep_sel_en(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
**作用：**使能SLP_SEL在轻度睡眠中自动改变GPIO状态

### gpio_sleep_sel_dis
**函数原型：**
```c
esp_err_t gpio_sleep_sel_dis(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
**作用：**禁用SLP_SEL在轻度睡眠中自动改变GPIO状态

### gpio_sleep_set_direction
**函数原型：**
```c
esp_err_t gpio_sleep_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
```
**参数：**
gpio_num：GPIO引脚编号
mode：GPIO方向
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置GPIO睡眠时方向

### gpio_sleep_set_pull_mode
**函数原型：**
```c
esp_err_t gpio_sleep_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);
```
**参数：**
gpio_num：GPIO引脚编号
pull：GPIO上下拉模式
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**配置GPIO睡眠时上下拉电阻

## 9. 调试和诊断函数

### gpio_dump_io_configuration
**函数原型：**
```c
esp_err_t gpio_dump_io_configuration(FILE *out_stream, uint64_t io_bit_mask);
```
**参数：**
out_stream：输出流（如stdout）
io_bit_mask：GPIO位掩码，每位对应一个GPIO
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**输出GPIO配置信息到控制台

### gpio_get_io_config
**函数原型：**
```c
esp_err_t gpio_get_io_config(gpio_num_t gpio_num, gpio_io_config_t *out_io_config);
```
**参数：**
gpio_num：GPIO引脚编号
out_io_config：指向保存特定GPIO配置的结构体指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**获取GPIO当前配置

## 10. RTC GPIO函数

### rtc_gpio_is_valid_gpio
**函数原型：**
```c
bool rtc_gpio_is_valid_gpio(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
true：该GPIO是有效的RTC GPIO
false：该GPIO不是RTC GPIO
**作用：**判断指定的GPIO是否为有效的RTC GPIO

### rtc_io_number_get
**函数原型：**
```c
int rtc_io_number_get(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
>=0：RTC GPIO的索引号
-1：该GPIO不是RTC GPIO
**作用：**通过GPIO编号获取RTC GPIO索引号

### rtc_gpio_init
**函数原型：**
```c
esp_err_t rtc_gpio_init(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**初始化GPIO为RTC GPIO，路由到RTC IO MUX

### rtc_gpio_deinit
**函数原型：**
```c
esp_err_t rtc_gpio_deinit(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**反初始化RTC GPIO，路由回IO MUX

### rtc_gpio_get_level
**函数原型：**
```c
uint32_t rtc_gpio_get_level(gpio_num_t gpio_num);
```
**参数：**
gpio_num：RTC GPIO引脚编号
**返回值：**
1：高电平
0：低电平
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**获取RTC GPIO输入电平

### rtc_gpio_set_level
**函数原型：**
```c
esp_err_t rtc_gpio_set_level(gpio_num_t gpio_num, uint32_t level);
```
**参数：**
gpio_num：RTC GPIO引脚编号
level：输出电平
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**设置RTC GPIO输出电平

### rtc_gpio_set_direction
**函数原型：**
```c
esp_err_t rtc_gpio_set_direction(gpio_num_t gpio_num, rtc_gpio_mode_t mode);
```
**参数：**
gpio_num：RTC GPIO引脚编号
mode：RTC GPIO方向
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**配置RTC GPIO方向

### rtc_gpio_set_direction_in_sleep
**函数原型：**
```c
esp_err_t rtc_gpio_set_direction_in_sleep(gpio_num_t gpio_num, rtc_gpio_mode_t mode);
```
**参数：**
gpio_num：RTC GPIO引脚编号
mode：RTC GPIO方向
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**设置RTC GPIO在深度睡眠模式或禁用睡眠状态时的方向

### rtc_gpio_pullup_en
**函数原型：**
```c
esp_err_t rtc_gpio_pullup_en(gpio_num_t gpio_num);
```
**参数：**
gpio_num：RTC GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**使能RTC GPIO上拉电阻

### rtc_gpio_pullup_dis
**函数原型：**
```c
esp_err_t rtc_gpio_pullup_dis(gpio_num_t gpio_num);
```
**参数：**
gpio_num：RTC GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**禁用RTC GPIO上拉电阻

### rtc_gpio_pulldown_en
**函数原型：**
```c
esp_err_t rtc_gpio_pulldown_en(gpio_num_t gpio_num);
```
**参数：**
gpio_num：RTC GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**使能RTC GPIO下拉电阻

### rtc_gpio_pulldown_dis
**函数原型：**
```c
esp_err_t rtc_gpio_pulldown_dis(gpio_num_t gpio_num);
```
**参数：**
gpio_num：RTC GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**禁用RTC GPIO下拉电阻

### rtc_gpio_set_drive_capability
**函数原型：**
```c
esp_err_t rtc_gpio_set_drive_capability(gpio_num_t gpio_num, gpio_drive_cap_t strength);
```
**参数：**
gpio_num：RTC GPIO引脚编号（仅支持输出GPIO）
strength：驱动能力
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**设置RTC GPIO驱动能力

### rtc_gpio_get_drive_capability
**函数原型：**
```c
esp_err_t rtc_gpio_get_drive_capability(gpio_num_t gpio_num, gpio_drive_cap_t* strength);
```
**参数：**
gpio_num：RTC GPIO引脚编号（仅支持输出GPIO）
strength：指向接受驱动能力值的指针
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**获取RTC GPIO驱动能力

### rtc_gpio_iomux_func_sel
**函数原型：**
```c
esp_err_t rtc_gpio_iomux_func_sel(gpio_num_t gpio_num, int func);
```
**参数：**
gpio_num：GPIO引脚编号
func：分配给引脚的功能
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**为RTC GPIO选择RTC IOMUX功能

### rtc_gpio_hold_en
**函数原型：**
```c
esp_err_t rtc_gpio_hold_en(gpio_num_t gpio_num);
```
**参数：**
gpio_num：RTC GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**使能RTC GPIO保持功能

### rtc_gpio_hold_dis
**函数原型：**
```c
esp_err_t rtc_gpio_hold_dis(gpio_num_t gpio_num);
```
**参数：**
gpio_num：RTC GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**禁用RTC GPIO保持功能

### rtc_gpio_force_hold_en_all
**函数原型：**
```c
esp_err_t rtc_gpio_force_hold_en_all(void);
```
**参数：**
无
**返回值：**
ESP_OK：操作成功
**作用：**使能所有RTC GPIO的强制保持信号

### rtc_gpio_force_hold_dis_all
**函数原型：**
```c
esp_err_t rtc_gpio_force_hold_dis_all(void);
```
**参数：**
无
**返回值：**
ESP_OK：操作成功
**作用：**禁用所有RTC GPIO的强制保持信号

### rtc_gpio_isolate
**函数原型：**
```c
esp_err_t rtc_gpio_isolate(gpio_num_t gpio_num);
```
**参数：**
gpio_num：RTC GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**断开RTC GPIO与内部电路的连接

### rtc_gpio_wakeup_enable
**函数原型：**
```c
esp_err_t rtc_gpio_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type);
```
**参数：**
gpio_num：GPIO引脚编号
intr_type：唤醒类型（GPIO_INTR_HIGH_LEVEL或GPIO_INTR_LOW_LEVEL）
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO，或中断类型不是GPIO_INTR_HIGH_LEVEL、GPIO_INTR_LOW_LEVEL之一
**作用：**使能特定GPIO的睡眠唤醒功能

### rtc_gpio_wakeup_disable
**函数原型：**
```c
esp_err_t rtc_gpio_wakeup_disable(gpio_num_t gpio_num);
```
**参数：**
gpio_num：GPIO引脚编号
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：该GPIO不是RTC GPIO
**作用：**禁用特定GPIO的睡眠唤醒功能

## 11. 专用GPIO函数

### dedic_gpio_new_bundle
**函数原型：**
```c
esp_err_t dedic_gpio_new_bundle(const dedic_gpio_bundle_config_t *config, dedic_gpio_bundle_handle_t *ret_bundle);
```
**参数：**
config：GPIO包配置
ret_bundle：返回的新创建GPIO包的句柄
**返回值：**
ESP_OK：创建GPIO包成功
ESP_ERR_INVALID_ARG：因无效参数创建GPIO包失败
ESP_ERR_NO_MEM：因内存不足创建GPIO包失败
ESP_ERR_NOT_FOUND：因没有足够连续的专用通道创建GPIO包失败
ESP_FAIL：因其他错误创建GPIO包失败
**作用：**创建GPIO包并返回句柄

### dedic_gpio_del_bundle
**函数原型：**
```c
esp_err_t dedic_gpio_del_bundle(dedic_gpio_bundle_handle_t bundle);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
**返回值：**
ESP_OK：销毁GPIO包成功
ESP_ERR_INVALID_ARG：因无效参数销毁GPIO包失败
ESP_FAIL：因其他错误销毁GPIO包失败
**作用：**销毁GPIO包

### dedic_gpio_get_out_mask
**函数原型：**
```c
esp_err_t dedic_gpio_get_out_mask(dedic_gpio_bundle_handle_t bundle, uint32_t *mask);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
mask：特定方向（输出）的返回掩码值
**返回值：**
ESP_OK：获取通道掩码成功
ESP_ERR_INVALID_ARG：因无效参数获取通道掩码失败
ESP_FAIL：因其他错误获取通道掩码失败
**作用：**获取分配的输出通道掩码

### dedic_gpio_get_in_mask
**函数原型：**
```c
esp_err_t dedic_gpio_get_in_mask(dedic_gpio_bundle_handle_t bundle, uint32_t *mask);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
mask：特定方向（输入）的返回掩码值
**返回值：**
ESP_OK：获取通道掩码成功
ESP_ERR_INVALID_ARG：因无效参数获取通道掩码失败
ESP_FAIL：因其他错误获取通道掩码失败
**作用：**获取分配的输入通道掩码

### dedic_gpio_get_out_offset
**函数原型：**
```c
esp_err_t dedic_gpio_get_out_offset(dedic_gpio_bundle_handle_t bundle, uint32_t *offset);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
offset：特定方向（输出）第一个通道的偏移值
**返回值：**
ESP_OK：获取通道偏移成功
ESP_ERR_INVALID_ARG：因无效参数获取通道偏移失败
ESP_FAIL：因其他错误获取通道偏移失败
**作用：**获取GPIO包的输出通道偏移

### dedic_gpio_get_in_offset
**函数原型：**
```c
esp_err_t dedic_gpio_get_in_offset(dedic_gpio_bundle_handle_t bundle, uint32_t *offset);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
offset：特定方向（输入）第一个通道的偏移值
**返回值：**
ESP_OK：获取通道偏移成功
ESP_ERR_INVALID_ARG：因无效参数获取通道偏移失败
ESP_FAIL：因其他错误获取通道偏移失败
**作用：**获取GPIO包的输入通道偏移

### dedic_gpio_bundle_write
**函数原型：**
```c
void dedic_gpio_bundle_write(dedic_gpio_bundle_handle_t bundle, uint32_t mask, uint32_t value);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
mask：给定包中要写入的GPIO掩码
value：要写入给定GPIO包的值，低位代表包中的低成员
**返回值：**
无
**作用：**向GPIO包写入值

### dedic_gpio_bundle_read_out
**函数原型：**
```c
uint32_t dedic_gpio_bundle_read_out(dedic_gpio_bundle_handle_t bundle);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
**返回值：**
从GPIO包输出的值，低位代表包中的低成员
**作用：**读取给定GPIO包的输出值

### dedic_gpio_bundle_read_in
**函数原型：**
```c
uint32_t dedic_gpio_bundle_read_in(dedic_gpio_bundle_handle_t bundle);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
**返回值：**
输入到GPIO包的值，低位代表包中的低成员
**作用：**读取给定GPIO包的输入值

### dedic_gpio_bundle_set_interrupt_and_callback
**函数原型：**
```c
esp_err_t dedic_gpio_bundle_set_interrupt_and_callback(dedic_gpio_bundle_handle_t bundle, uint32_t mask, dedic_gpio_intr_type_t intr_type, dedic_gpio_isr_callback_t cb_isr, void *cb_args);
```
**参数：**
bundle：从dedic_gpio_new_bundle返回的GPIO包句柄
mask：给定包中GPIO的掩码
intr_type：中断类型，设置为DEDIC_GPIO_INTR_NONE可禁用中断
cb_isr：回调函数，在ISR上下文中调用。这里的空指针将绕过回调
cb_args：传递给回调函数的用户定义参数
**返回值：**
ESP_OK：设置GPIO中断和回调函数成功
ESP_ERR_INVALID_ARG：因无效参数设置GPIO中断和回调函数失败
ESP_FAIL：因其他错误设置GPIO中断和回调函数失败
**作用：**为GPIO包设置中断和回调函数

## 12. GPIO ETM函数

### gpio_new_etm_event
**函数原型：**
```c
esp_err_t gpio_new_etm_event(const gpio_etm_event_config_t *config, esp_etm_event_handle_t *ret_event, ...);
```
**参数：**
config：GPIO ETM事件配置
ret_event：返回的ETM事件句柄
...：如果有其他返回的ETM事件句柄（返回事件句柄的顺序与gpio_etm_event_config_t中edges字段的数组顺序对齐）
**返回值：**
ESP_OK：创建ETM事件成功
ESP_ERR_INVALID_ARG：因无效参数创建ETM事件失败
ESP_ERR_NO_MEM：因内存不足创建ETM事件失败
ESP_ERR_NOT_FOUND：因所有事件都被使用且没有更多空闲事件创建ETM事件失败
ESP_FAIL：因其他原因创建ETM事件失败
**作用：**为GPIO外设创建ETM事件对象

### gpio_etm_event_bind_gpio
**函数原型：**
```c
esp_err_t gpio_etm_event_bind_gpio(esp_etm_event_handle_t event, int gpio_num);
```
**参数：**
event：由gpio_new_etm_event创建的ETM事件句柄
gpio_num：可以触发ETM事件的GPIO编号
**返回值：**
ESP_OK：为ETM事件设置GPIO成功
ESP_ERR_INVALID_ARG：因无效参数为ETM事件设置GPIO失败，例如GPIO不具备输入能力，ETM事件不是GPIO类型
ESP_FAIL：因其他原因为ETM事件设置GPIO失败
**作用：**将GPIO与ETM事件绑定

### gpio_new_etm_task
**函数原型：**
```c
esp_err_t gpio_new_etm_task(const gpio_etm_task_config_t *config, esp_etm_task_handle_t *ret_task, ...);
```
**参数：**
config：GPIO ETM任务配置
ret_task：返回的ETM任务句柄
...：如果有其他返回的ETM任务句柄（返回任务句柄的顺序与gpio_etm_task_config_t中actions字段的数组顺序对齐）
**返回值：**
ESP_OK：创建ETM任务成功
ESP_ERR_INVALID_ARG：因无效参数创建ETM任务失败
ESP_ERR_NO_MEM：因内存不足创建ETM任务失败
ESP_ERR_NOT_FOUND：因所有任务都被使用且没有更多空闲任务创建ETM任务失败
ESP_FAIL：因其他原因创建ETM任务失败
**作用：**为GPIO外设创建ETM任务对象

### gpio_etm_task_add_gpio
**函数原型：**
```c
esp_err_t gpio_etm_task_add_gpio(esp_etm_task_handle_t task, int gpio_num);
```
**参数：**
task：由gpio_new_etm_task创建的ETM任务句柄
gpio_num：可以被ETM任务控制的GPIO编号
**返回值：**
ESP_OK：向ETM任务添加GPIO成功
ESP_ERR_INVALID_ARG：因无效参数向ETM任务添加GPIO失败，例如GPIO不具备输出能力，ETM任务不是GPIO类型
ESP_ERR_INVALID_STATE：因GPIO已被其他ETM任务使用而向ETM任务添加GPIO失败
ESP_FAIL：因其他原因向ETM任务添加GPIO失败
**作用：**向ETM任务添加GPIO

### gpio_etm_task_rm_gpio
**函数原型：**
```c
esp_err_t gpio_etm_task_rm_gpio(esp_etm_task_handle_t task, int gpio_num);
```
**参数：**
task：由gpio_new_etm_task创建的ETM任务句柄
gpio_num：要从ETM任务中移除的GPIO编号
**返回值：**
ESP_OK：从ETM任务移除GPIO成功
ESP_ERR_INVALID_ARG：因无效参数从ETM任务移除GPIO失败
ESP_ERR_INVALID_STATE：因GPIO不受此ETM任务控制而从ETM任务移除GPIO失败
ESP_FAIL：因其他原因从ETM任务移除GPIO失败
**作用：**从ETM任务中移除GPIO

## 13. GPIO滤波器函数

### gpio_new_pin_glitch_filter
**函数原型：**
```c
esp_err_t gpio_new_pin_glitch_filter(const gpio_pin_glitch_filter_config_t *config, gpio_glitch_filter_handle_t *ret_filter);
```
**参数：**
config：毛刺滤波器配置
ret_filter：返回的毛刺滤波器句柄
**返回值：**
ESP_OK：创建引脚毛刺滤波器成功
ESP_ERR_INVALID_ARG：因无效参数创建引脚毛刺滤波器失败（例如错误的GPIO编号）
ESP_ERR_NO_MEM：因内存不足创建引脚毛刺滤波器失败
ESP_FAIL：因其他错误创建引脚毛刺滤波器失败
**作用：**创建引脚毛刺滤波器

### gpio_new_flex_glitch_filter
**函数原型：**
```c
esp_err_t gpio_new_flex_glitch_filter(const gpio_flex_glitch_filter_config_t *config, gpio_glitch_filter_handle_t *ret_filter);
```
**参数：**
config：毛刺滤波器配置
ret_filter：返回的毛刺滤波器句柄
**返回值：**
ESP_OK：分配灵活毛刺滤波器成功
ESP_ERR_INVALID_ARG：因无效参数分配灵活毛刺滤波器失败（例如错误的GPIO编号，滤波器参数超出范围）
ESP_ERR_NO_MEM：因内存不足分配灵活毛刺滤波器失败
ESP_ERR_NOT_FOUND：因底层硬件资源用完而分配灵活毛刺滤波器失败
ESP_FAIL：因其他错误分配灵活毛刺滤波器失败
**作用：**分配灵活毛刺滤波器

### gpio_del_glitch_filter
**函数原型：**
```c
esp_err_t gpio_del_glitch_filter(gpio_glitch_filter_handle_t filter);
```
**参数：**
filter：从gpio_new_flex_glitch_filter或gpio_new_pin_glitch_filter返回的毛刺滤波器句柄
**返回值：**
ESP_OK：删除毛刺滤波器成功
ESP_ERR_INVALID_ARG：因无效参数删除毛刺滤波器失败
ESP_ERR_INVALID_STATE：因毛刺滤波器仍在工作而删除毛刺滤波器失败
ESP_FAIL：因其他错误删除毛刺滤波器失败
**作用：**删除毛刺滤波器

### gpio_glitch_filter_enable
**函数原型：**
```c
esp_err_t gpio_glitch_filter_enable(gpio_glitch_filter_handle_t filter);
```
**参数：**
filter：从gpio_new_flex_glitch_filter或gpio_new_pin_glitch_filter返回的毛刺滤波器句柄
**返回值：**
ESP_OK：使能毛刺滤波器成功
ESP_ERR_INVALID_ARG：因无效参数使能毛刺滤波器失败
ESP_ERR_INVALID_STATE：因毛刺滤波器已使能而使能毛刺滤波器失败
ESP_FAIL：因其他错误使能毛刺滤波器失败
**作用：**使能毛刺滤波器

### gpio_glitch_filter_disable
**函数原型：**
```c
esp_err_t gpio_glitch_filter_disable(gpio_glitch_filter_handle_t filter);
```
**参数：**
filter：从gpio_new_flex_glitch_filter或gpio_new_pin_glitch_filter返回的毛刺滤波器句柄
**返回值：**
ESP_OK：禁用毛刺滤波器成功
ESP_ERR_INVALID_ARG：因无效参数禁用毛刺滤波器失败
ESP_ERR_INVALID_STATE：因毛刺滤波器尚未使能而禁用毛刺滤波器失败
ESP_FAIL：因其他错误禁用毛刺滤波器失败
**作用：**禁用毛刺滤波器

## 14. 低功耗GPIO函数

### lp_gpio_connect_in_signal
**函数原型：**
```c
esp_err_t lp_gpio_connect_in_signal(gpio_num_t gpio_num, uint32_t signal_idx, bool inv);
```
**参数：**
gpio_num：GPIO编号，特别地，LP_GPIO_MATRIX_CONST_ZERO_INPUT意味着将逻辑0连接到信号，LP_GPIO_MATRIX_CONST_ONE_INPUT意味着将逻辑1连接到信号
signal_idx：LP外设信号索引（标记为输入属性）
inv：RTC(LP) GPIO输入是否要反相
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**通过外设信号连接RTC(LP) GPIO输入，该信号标记为输入属性

### lp_gpio_connect_out_signal
**函数原型：**
```c
esp_err_t lp_gpio_connect_out_signal(gpio_num_t gpio_num, uint32_t signal_idx, bool out_inv, bool out_en_inv);
```
**参数：**
gpio_num：GPIO编号
signal_idx：LP外设信号索引（标记为输入属性），特别地，SIG_LP_GPIO_OUT_IDX意味着断开RTC(LP) GPIO和其他外设的连接。只有RTC GPIO驱动程序可以控制输出电平
out_inv：信号是否要反相
out_en_inv：输出使能控制是否反相
**返回值：**
ESP_OK：操作成功
ESP_ERR_INVALID_ARG：参数错误
**作用：**连接标记为输出属性的外设信号与RTC(LP) GPIO
