# ESP32 I2C Master API 完整函数参考

## 0. I2C Master主要数据结构

### i2c_master_bus_config_t

I2C主线总线配置结构体，用于配置I2C主线总线的所有参数：

```c
typedef struct {
    i2c_port_num_t i2c_port;              // I2C端口号
    gpio_num_t sda_io_num;                // SDA引脚号
    gpio_num_t scl_io_num;                // SCL引脚号
    union {
        i2c_clock_source_t clk_source;        // 时钟源
#if SOC_LP_I2C_SUPPORTED
        lp_i2c_clock_source_t lp_source_clk;  // 低功耗时钟源
#endif
    };
    uint8_t glitch_ignore_cnt;            // 毛刺忽略计数
    int intr_priority;                    // 中断优先级
    size_t trans_queue_depth;             // 传输队列深度
    struct {
        uint32_t enable_internal_pullup: 1;  // 启用内部上拉
        uint32_t allow_pd: 1;                // 允许掉电
    } flags;                              // 配置标志
} i2c_master_bus_config_t;
```

**详细成员说明：**

- **`i2c_port`** (类型: `i2c_port_num_t`)
  - **功能**: 指定使用的I2C端口号
  - **取值范围**: `I2C_NUM_0` 到 `I2C_NUM_MAX-1`，或 `-1` 自动选择
  - **说明**: `-1` 表示自动选择可用端口（不包括LP I2C实例）
  - **注意事项**: 不同芯片支持的I2C端口数量不同

- **`sda_io_num`** (类型: `gpio_num_t`)
  - **功能**: 指定I2C SDA（数据线）的GPIO引脚号
  - **取值范围**: 有效的GPIO引脚编号
  - **硬件要求**: 需要外部上拉电阻（通常4.7kΩ或10kΩ）
  - **注意事项**: 必须是支持输入输出的GPIO引脚

- **`scl_io_num`** (类型: `gpio_num_t`)
  - **功能**: 指定I2C SCL（时钟线）的GPIO引脚号
  - **取值范围**: 有效的GPIO引脚编号
  - **硬件要求**: 需要外部上拉电阻（通常4.7kΩ或10kΩ）
  - **注意事项**: 必须是支持输入输出的GPIO引脚

- **`clk_source`** (类型: `i2c_clock_source_t`)
  - **功能**: 选择I2C主线的时钟源
  - **可选值**:
    - `I2C_CLK_SRC_DEFAULT`: 默认时钟源
    - `I2C_CLK_SRC_APB`: APB时钟
    - `I2C_CLK_SRC_XTAL`: 外部晶振时钟
  - **影响**: 影响I2C总线的时钟精度和功耗

- **`lp_source_clk`** (类型: `lp_i2c_clock_source_t`, 仅支持低功耗I2C的芯片)
  - **功能**: 选择低功耗I2C的时钟源
  - **应用场景**: 用于低功耗模式下的I2C通信

- **`glitch_ignore_cnt`** (类型: `uint8_t`)
  - **功能**: 设置毛刺过滤计数器
  - **取值范围**: 0-255
  - **默认值**: 通常设置为7
  - **作用**: 过滤掉持续时间小于此值的信号毛刺（单位：I2C模块时钟周期）

- **`intr_priority`** (类型: `int`)
  - **功能**: 设置I2C中断优先级
  - **取值范围**: 0表示使用默认优先级，其他值为1-7
  - **默认值**: 0（驱动会选择默认优先级1,2,3）
  - **注意事项**: 优先级数值越小，优先级越高

- **`trans_queue_depth`** (类型: `size_t`)
  - **功能**: 设置内部传输队列深度
  - **计算建议**: 通常设置为 max_device_num × per_transaction
  - **应用**: 增加此值可支持更多并发传输挂起在后台
  - **仅对异步传输有效**: 同步传输不使用此队列

- **`flags.enable_internal_pullup`** (类型: `uint32_t`, 1位)
  - **功能**: 启用内部上拉电阻
  - **取值**: 0-禁用，1-启用
  - **注意事项**: 内部上拉电阻较弱，高速频率下建议使用外部上拉
  - **应用场景**: 临时测试或低速通信时使用

- **`flags.allow_pd`** (类型: `uint32_t`, 1位)
  - **功能**: 允许在轻度睡眠时掉电
  - **取值**: 0-不允许，1-允许
  - **作用**: 启用后驱动会在睡眠前备份/恢复I2C寄存器
  - **权衡**: 可以节省功耗，但会消耗更多RAM

### i2c_device_config_t

I2C设备配置结构体，用于配置连接到I2C总线的特定设备：

```c
typedef struct {
    i2c_addr_bit_len_t dev_addr_length;         // 设备地址长度
    uint16_t device_address;                    // 设备地址
    uint32_t scl_speed_hz;                      // SCL时钟频率
    uint32_t scl_wait_us;                      // SCL等待超时
    struct {
        uint32_t disable_ack_check: 1;         // 禁用ACK检查
    } flags;                                    // 设备配置标志
} i2c_device_config_t;
```

**详细成员说明：**

- **`dev_addr_length`** (类型: `i2c_addr_bit_len_t`)
  - **功能**: 选择从设备的地址长度
  - **可选值**:
    - `I2C_ADDR_BIT_LEN_7`: 7位地址模式（标准模式）
    - `I2C_ADDR_BIT_LEN_10`: 10位地址模式（扩展模式）
  - **应用**: 大部分I2C设备使用7位地址

- **`device_address`** (类型: `uint16_t`)
  - **功能**: I2C设备的原始地址（不包含读写位）
  - **取值范围**: 7位地址：0x00-0x7F，10位地址：0x000-0x3FF
  - **特殊值**: `I2C_DEVICE_ADDRESS_NOT_USED` (0xFFFF) 表示跳过地址配置
  - **注意事项**: 这是设备的实际地址，不包含最低位的读写标识位

- **`scl_speed_hz`** (类型: `uint32_t`)
  - **功能**: 设置I2C SCL线频率
  - **常用值**:
    - 100000: 标准模式 (100kHz)
    - 400000: 快速模式 (400kHz)
    - 1000000: 快速模式+ (1MHz)
    - 3400000: 高速模式 (3.4MHz，部分芯片支持)
  - **注意事项**: 实际频率可能因硬件限制略有差异

- **`scl_wait_us`** (类型: `uint32_t`)
  - **功能**: SCL时钟延展超时值（微秒）
  - **取值**: 0表示使用默认寄存器值，其他值为具体超时时间
  - **作用**: 处理从设备时钟延展或干扰的超时保护
  - **建议**: 设置足够大的值以正确处理时钟延展

- **`flags.disable_ack_check`** (类型: `uint32_t`, 1位)
  - **功能**: 禁用ACK检查
  - **取值**: 0-启用ACK检查，1-禁用ACK检查
  - **默认**: 通常启用ACK检查，检测到NACK时停止传输并返回错误
  - **应用场景**: 某些特殊协议或调试时可能需要禁用

### i2c_operation_job_t

I2C操作任务结构体，用于定义I2C主设备事务中的单个操作：

```c
typedef struct {
    i2c_master_command_t command;  // I2C命令类型
    union {
        struct {
            bool ack_check;        // WRITE操作的ACK检查
            uint8_t *data;         // 写入数据指针
            size_t total_bytes;    // 总写入字节数
        } write;
        struct {
            i2c_ack_value_t ack_value; // READ操作后的ACK值
            uint8_t *data;              // 读取数据缓冲区指针
            size_t total_bytes;         // 总读取字节数
        } read;
    };
} i2c_operation_job_t;
```

**详细成员说明：**

- **`command`** (类型: `i2c_master_command_t`)
  - **功能**: 指示操作类型
  - **可选值**:
    - `I2C_MASTER_CMD_START`: 发送START条件
    - `I2C_MASTER_CMD_WRITE`: 写入数据
    - `I2C_MASTER_CMD_READ`: 读取数据
    - `I2C_MASTER_CMD_STOP`: 发送STOP条件
  - **序列**: 通常按START->WRITE/READ->STOP顺序组合

- **`write.ack_check`** (类型: `bool`)
  - **功能**: WRITE操作期间是否启用ACK检查
  - **取值**: true-启用，false-禁用
  - **建议**: 通常启用以检测传输错误

- **`write.data`** (类型: `uint8_t *`)
  - **功能**: 指向要写入数据的指针
  - **要求**: 数据缓冲区必须保持有效直到操作完成
  - **注意事项**: 异步操作时缓冲区生命周期特别重要

- **`write.total_bytes`** (类型: `size_t`)
  - **功能**: 要写入的总字节数
  - **限制**: 受硬件FIFO大小和驱动实现限制

- **`read.ack_value`** (类型: `i2c_ack_value_t`)
  - **功能**: 读取后发送的ACK值
  - **可选值**:
    - `I2C_ACK_VAL`: 发送ACK（继续读取）
    - `I2C_NACK_VAL`: 发送NACK（结束读取）
  - **规则**: 如果下一个操作是STOP，必须设置为`I2C_NACK_VAL`

- **`read.data`** (类型: `uint8_t *`)
  - **功能**: 用于存储从总线读取数据的缓冲区指针
  - **要求**: 缓冲区大小必须足够存储total_bytes字节

- **`read.total_bytes`** (类型: `size_t`)
  - **功能**: 要读取的总字节数
  - **限制**: 受硬件FIFO大小限制

### i2c_master_transmit_multi_buffer_info_t

I2C主设备多缓冲区传输信息结构体：

```c
typedef struct {
    uint8_t *write_buffer;               // 写入缓冲区指针
    size_t buffer_size;                  // 数据大小
} i2c_master_transmit_multi_buffer_info_t;
```

**详细成员说明：**

- **`write_buffer`** (类型: `uint8_t *`)
  - **功能**: 指向要写入数据的缓冲区
  - **要求**: 缓冲区必须保持有效直到传输完成

- **`buffer_size`** (类型: `size_t`)
  - **功能**: 指定缓冲区中数据的大小
  - **单位**: 字节

### i2c_master_event_callbacks_t

I2C主设备事件回调函数组结构体：

```c
typedef struct {
    i2c_master_callback_t on_trans_done;  // 传输完成回调
} i2c_master_event_callbacks_t;
```

**详细成员说明：**

- **`on_trans_done`** (类型: `i2c_master_callback_t`)
  - **功能**: 传输完成时的回调函数
  - **运行环境**: 在ISR上下文中运行
  - **注意事项**: 当启用CONFIG_I2C_ISR_IRAM_SAFE时，回调函数必须放在IRAM中
  - **用途**: 可用于异步传输的状态监控或执行其他小任务

## 1. I2C Master总线管理函数

### i2c_new_master_bus
**函数原型：**
```c
esp_err_t i2c_new_master_bus(const i2c_master_bus_config_t *bus_config, i2c_master_bus_handle_t *ret_bus_handle);
```
**参数：**
- `bus_config`：指向I2C主线总线配置信息的结构体指针
- `ret_bus_handle`：返回的I2C总线句柄指针

**返回值：**
- `ESP_OK`：I2C主线总线初始化成功
- `ESP_ERR_INVALID_ARG`：I2C总线初始化失败，参数无效
- `ESP_ERR_NO_MEM`：创建I2C总线失败，内存不足
- `ESP_ERR_NOT_FOUND`：没有更多可用的总线

**作用：**分配一个I2C主线总线，配置GPIO引脚、时钟源、中断等参数，返回总线句柄供后续操作使用

### i2c_del_master_bus
**函数原型：**
```c
esp_err_t i2c_del_master_bus(i2c_master_bus_handle_t bus_handle);
```
**参数：**
- `bus_handle`：要删除的I2C总线句柄

**返回值：**
- `ESP_OK`：删除I2C总线成功
- `ESP_ERR_INVALID_ARG`：参数无效
- `ESP_FAIL`：删除失败

**作用：**取消初始化I2C主线总线并删除句柄，释放相关资源

### i2c_master_bus_reset
**函数原型：**
```c
esp_err_t i2c_master_bus_reset(i2c_master_bus_handle_t bus_handle);
```
**参数：**
- `bus_handle`：要重置的I2C总线句柄

**返回值：**
- `ESP_OK`：重置成功
- `ESP_ERR_INVALID_ARG`：I2C主线总线句柄未初始化
- 其他值：重置失败

**作用：**重置I2C主线总线，清除总线状态，通常用于从错误状态恢复

### i2c_master_get_bus_handle
**函数原型：**
```c
esp_err_t i2c_master_get_bus_handle(i2c_port_num_t port_num, i2c_master_bus_handle_t *ret_handle);
```
**参数：**
- `port_num`：要获取句柄的I2C端口号
- `ret_handle`：用于存储检索到的句柄的指针

**返回值：**
- `ESP_OK`：成功检索句柄
- `ESP_ERR_INVALID_ARG`：无效参数，如无效端口号
- `ESP_ERR_INVALID_STATE`：无效状态，如I2C端口未初始化

**作用：**检索指定I2C端口号的I2C主线总线句柄，句柄必须已经初始化

### i2c_master_isr_handler_default
**函数原型：**
```c
static void i2c_master_isr_handler_default(void *arg);
```
**参数：**
- `arg`：传递给ISR的参数，通常是I2C主设备总线句柄

**作用：**I2C主设备的默认中断服务程序，处理各种I2C中断事件

**详细功能：**
- **中断状态检查**: 获取并清除中断标志位
- **错误处理**: 处理NACK、超时、仲裁丢失等错误
- **传输完成**: 处理主设备传输完成中断
- **接收数据处理**: 处理接收FIFO数据
- **异步操作支持**: 支持异步传输模式的中断处理

**中断类型处理：**
- `I2C_LL_INTR_NACK`: 处理从设备NACK响应
- `I2C_LL_INTR_TIMEOUT`: 处理超时错误
- `I2C_LL_INTR_ARBITRATION`: 处理总线仲裁丢失
- `I2C_LL_INTR_MST_COMPLETE`: 处理主设备传输完成

### i2c_isr_receive_handler
**函数原型：**
```c
I2C_MASTER_ISR_ATTR static void i2c_isr_receive_handler(i2c_master_bus_t *i2c_master);
```
**参数：**
- `i2c_master`：I2C主设备总线结构体指针

**作用：**处理I2C主设备接收数据的中断服务程序

**详细功能：**
- **接收状态检查**: 检查当前是否处于读取状态
- **FIFO数据读取**: 从硬件接收FIFO读取数据
- **数据缓冲**: 将接收到的数据存储到用户缓冲区
- **读取计数管理**: 更新已读取的字节计数
- **读取完成处理**: 处理读取操作完成的后续操作

**读取流程：**
1. 检查接收状态和数据可用性
2. 从硬件FIFO读取数据到缓冲区
3. 更新读取位置和剩余长度
4. 处理读取完成事件

### i2c_param_master_config
**函数原型：**
```c
static esp_err_t i2c_param_master_config(i2c_bus_handle_t handle, const i2c_master_bus_config_t *i2c_conf);
```
**参数：**
- `handle`：I2C总线句柄
- `i2c_conf`：I2C主设备总线配置结构体指针

**返回值：**
- `ESP_OK`：参数配置成功
- `ESP_ERR_INVALID_ARG`：参数无效

**作用：**配置I2C主设备的参数，包括时钟、引脚、中断等设置

**详细功能：**
- **引脚配置**: 设置SDA和SCL引脚
- **时钟源选择**: 配置I2C时钟源
- **中断配置**: 设置中断优先级和处理程序
- **硬件参数**: 配置FIFO阈值、超时时间等
- **电源管理**: 配置电源管理锁

**配置项目：**
- GPIO引脚模式和上拉设置
- 时钟频率和分频器配置
- 中断使能和优先级设置
- FIFO水位标记设置

### i2c_master_bus_destroy
**函数原型：**
```c
static esp_err_t i2c_master_bus_destroy(i2c_master_bus_handle_t bus_handle);
```
**参数：**
- `bus_handle`：要销毁的I2C主设备总线句柄

**返回值：**
- `ESP_OK`：总线销毁成功
- 其他错误码：销毁失败

**作用：**销毁I2C主设备总线，释放所有相关资源

**详细功能：**
- **中断清理**: 禁用并释放中断资源
- **引脚释放**: 恢复GPIO引脚到默认状态
- **内存释放**: 释放总线结构体和相关内存
- **队列清理**: 清理事件队列和信号量
- **时钟关闭**: 关闭I2C模块时钟

**清理顺序：**
1. 禁用所有中断
2. 停止正在进行的传输
3. 释放硬件资源
4. 清理软件资源
5. 释放内存

## 2. I2C设备管理函数

### i2c_master_bus_add_device
**函数原型：**
```c
esp_err_t i2c_master_bus_add_device(i2c_master_bus_handle_t bus_handle, const i2c_device_config_t *dev_config, i2c_master_dev_handle_t *ret_handle);
```
**参数：**
- `bus_handle`：I2C总线句柄
- `dev_config`：设备配置信息
- `ret_handle`：返回的设备句柄指针

**返回值：**
- `ESP_OK`：创建I2C主设备成功
- `ESP_ERR_INVALID_ARG`：I2C总线初始化失败，参数无效
- `ESP_ERR_NO_MEM`：创建I2C总线失败，内存不足

**作用：**在I2C主线总线上添加一个设备，配置设备地址、时钟频率等参数

### i2c_master_bus_rm_device
**函数原型：**
```c
esp_err_t i2c_master_bus_rm_device(i2c_master_dev_handle_t handle);
```
**参数：**
- `handle`：要删除的I2C设备句柄

**返回值：**
- `ESP_OK`：设备成功删除

**作用：**从I2C主线总线中删除设备，释放设备相关资源

### i2c_master_device_change_address
**函数原型：**
```c
esp_err_t i2c_master_device_change_address(i2c_master_dev_handle_t i2c_dev, uint16_t new_device_address, int timeout_ms);
```
**参数：**
- `i2c_dev`：I2C设备句柄
- `new_device_address`：新的设备地址
- `timeout_ms`：地址更改操作的超时时间（毫秒）

**返回值：**
- `ESP_OK`：地址成功更改
- `ESP_ERR_INVALID_ARG`：无效参数（如NULL句柄或无效地址）
- `ESP_ERR_TIMEOUT`：操作超时

**作用：**在运行时更改现有I2C设备句柄的设备地址，用于支持动态地址分配的设备

## 3. I2C数据传输函数

### i2c_master_transmit
**函数原型：**
```c
esp_err_t i2c_master_transmit(i2c_master_dev_handle_t i2c_dev, const uint8_t *write_buffer, size_t write_size, int xfer_timeout_ms);
```
**参数：**
- `i2c_dev`：I2C主设备句柄
- `write_buffer`：要在I2C总线上发送的数据字节
- `write_size`：写缓冲区的大小（字节）
- `xfer_timeout_ms`：等待超时时间（毫秒），-1表示永远等待

**返回值：**
- `ESP_OK`：I2C主线传输成功
- `ESP_ERR_INVALID_ARG`：I2C主线传输参数无效
- `ESP_ERR_TIMEOUT`：操作超时，因为总线忙或硬件崩溃

**作用：**在I2C总线上执行写事务，事务将继续进行直到完成或达到超时

### i2c_master_multi_buffer_transmit
**函数原型：**
```c
esp_err_t i2c_master_multi_buffer_transmit(i2c_master_dev_handle_t i2c_dev, i2c_master_transmit_multi_buffer_info_t *buffer_info_array, size_t array_size, int xfer_timeout_ms);
```
**参数：**
- `i2c_dev`：I2C主设备句柄
- `buffer_info_array`：缓冲区信息数组指针
- `array_size`：缓冲区信息数组的大小
- `xfer_timeout_ms`：等待超时时间（毫秒），-1表示永远等待

**返回值：**
- `ESP_OK`：I2C主线传输成功
- `ESP_ERR_INVALID_ARG`：I2C主线传输参数无效
- `ESP_ERR_TIMEOUT`：操作超时，因为总线忙或硬件崩溃

**作用：**通过I2C总线传输多个数据缓冲区，适用于需要发送多个不连续数据块的场景

### i2c_master_receive
**函数原型：**
```c
esp_err_t i2c_master_receive(i2c_master_dev_handle_t i2c_dev, uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
```
**参数：**
- `i2c_dev`：I2C主设备句柄
- `read_buffer`：从I2C总线接收数据字节的缓冲区
- `read_size`：读缓冲区的大小（字节）
- `xfer_timeout_ms`：等待超时时间（毫秒），-1表示永远等待

**返回值：**
- `ESP_OK`：I2C主线接收成功
- `ESP_ERR_INVALID_ARG`：I2C主线接收参数无效
- `ESP_ERR_TIMEOUT`：操作超时，因为总线忙或硬件崩溃

**作用：**在I2C总线上执行读事务，从指定设备读取数据

### i2c_master_transmit_receive
**函数原型：**
```c
esp_err_t i2c_master_transmit_receive(i2c_master_dev_handle_t i2c_dev, const uint8_t *write_buffer, size_t write_size, uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
```
**参数：**
- `i2c_dev`：I2C主设备句柄
- `write_buffer`：要在I2C总线上发送的数据字节
- `write_size`：写缓冲区的大小（字节）
- `read_buffer`：从I2C总线接收数据字节的缓冲区
- `read_size`：读缓冲区的大小（字节）
- `xfer_timeout_ms`：等待超时时间（毫秒），-1表示永远等待

**返回值：**
- `ESP_OK`：I2C主线传输接收成功
- `ESP_ERR_INVALID_ARG`：I2C主线传输参数无效
- `ESP_ERR_TIMEOUT`：操作超时，因为总线忙或硬件崩溃

**作用：**在I2C总线上执行写-读事务，先写入数据然后从设备读取响应数据

## 4. I2C高级操作函数

### i2c_master_probe
**函数原型：**
```c
esp_err_t i2c_master_probe(i2c_master_bus_handle_t bus_handle, uint16_t address, int xfer_timeout_ms);
```
**参数：**
- `bus_handle`：I2C主设备句柄
- `address`：要探测的I2C设备地址
- `xfer_timeout_ms`：等待超时时间（毫秒），-1表示永远等待（此函数不推荐）

**返回值：**
- `ESP_OK`：I2C设备探测成功
- `ESP_ERR_NOT_FOUND`：I2C探测失败，未找到指定地址的设备
- `ESP_ERR_TIMEOUT`：操作超时，因为总线忙或硬件崩溃

**作用：**探测I2C地址，如果地址正确且收到ACK，此函数将返回ESP_OK

**注意事项：**
- 必须连接上拉电阻到SCL和SDA引脚
- 如果没有合适的电阻，可以使用`flags.enable_internal_pullup`
- 探测原理是发送设备地址加写命令，根据ACK/NACK响应判断设备是否存在

### i2c_master_execute_defined_operations
**函数原型：**
```c
esp_err_t i2c_master_execute_defined_operations(i2c_master_dev_handle_t i2c_dev, i2c_operation_job_t *i2c_operation, size_t operation_list_num, int xfer_timeout_ms);
```
**参数：**
- `i2c_dev`：I2C主设备句柄
- `i2c_operation`：用户定义的I2C操作任务数组指针
- `operation_list_num`：`i2c_operation`数组中的操作数量
- `xfer_timeout_ms`：传输超时时间（毫秒）

**返回值：**
- `ESP_OK`：事务成功完成
- `ESP_ERR_INVALID_ARG`：一个或多个参数无效
- `ESP_ERR_TIMEOUT`：事务超时
- `ESP_FAIL`：事务期间的其他错误

**作用：**执行一系列预定义的I2C操作，如start、write、read和stop，根据用户定义的操作任务数组顺序执行

**注意事项：**
- 如果下一个操作是STOP命令，READ操作中的`ack_value`字段必须设置为`I2C_NACK_VAL`

## 5. I2C回调和异步操作函数

### i2c_master_register_event_callbacks
**函数原型：**
```c
esp_err_t i2c_master_register_event_callbacks(i2c_master_dev_handle_t i2c_dev, const i2c_master_event_callbacks_t *cbs, void *user_data);
```
**参数：**
- `i2c_dev`：I2C主设备句柄
- `cbs`：回调函数组
- `user_data`：用户数据，将直接传递给回调函数

**返回值：**
- `ESP_OK`：设置I2C事务回调成功
- `ESP_ERR_INVALID_ARG`：设置I2C事务回调失败，参数无效
- `ESP_FAIL`：设置I2C事务回调失败，其他错误

**作用：**为主设备注册I2C事务回调，用于异步操作状态监控

**注意事项：**
- 可以通过将`cbs`结构中的回调成员设置为NULL来注销先前注册的回调
- 当启用CONFIG_I2C_ISR_IRAM_SAFE时，回调本身和其调用的函数应放在IRAM中
- 在同一总线上，只有一个设备可用于执行异步操作

### i2c_master_bus_wait_all_done
**函数原型：**
```c
esp_err_t i2c_master_bus_wait_all_done(i2c_master_bus_handle_t bus_handle, int timeout_ms);
```
**参数：**
- `bus_handle`：I2C总线句柄
- `timeout_ms`：等待超时时间（毫秒），-1表示永远等待

**返回值：**
- `ESP_OK`：刷新事务成功
- `ESP_ERR_INVALID_ARG`：刷新事务失败，参数无效
- `ESP_ERR_TIMEOUT`：刷新事务失败，超时
- `ESP_FAIL`：刷新事务失败，其他错误

**作用：**等待所有挂起的I2C事务完成，确保所有异步操作都已处理完毕
