# ESP32 I2C Slave API 完整函数参考

## 0. I2C Slave主要数据结构

### i2c_slave_config_t

I2C从设备配置结构体，用于配置I2C从设备的所有参数：

```c
typedef struct {
    i2c_port_num_t i2c_port;                 // I2C端口号
    gpio_num_t sda_io_num;                   // SDA引脚号
    gpio_num_t scl_io_num;                   // SCL引脚号
    i2c_clock_source_t clk_source;           // 时钟源
    uint32_t send_buf_depth;                 // 发送缓冲区深度
    uint32_t receive_buf_depth;              // 接收缓冲区深度
    uint16_t slave_addr;                     // I2C从设备地址
    i2c_addr_bit_len_t addr_bit_len;         // 地址位长度
    int intr_priority;                       // 中断优先级
    struct {
        uint32_t allow_pd: 1;                // 允许掉电
        uint32_t enable_internal_pullup: 1;  // 启用内部上拉
#if SOC_I2C_SLAVE_SUPPORT_BROADCAST
        uint32_t broadcast_en: 1;            // 启用广播模式
#endif
    } flags;                                 // 配置标志
} i2c_slave_config_t;
```

**详细成员说明：**

- **`i2c_port`** (类型: `i2c_port_num_t`)
  - **功能**: 指定使用的I2C端口号
  - **取值范围**: `I2C_NUM_0` 到 `SOC_HP_I2C_NUM-1`，或 `-1` 自动选择
  - **限制**: LP I2C 不支持从设备模式
  - **注意事项**: `-1` 表示自动选择可用的HP I2C端口

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
  - **功能**: 选择I2C从设备的时钟源
  - **可选值**:
    - `I2C_CLK_SRC_DEFAULT`: 默认时钟源
    - `I2C_CLK_SRC_APB`: APB时钟
    - `I2C_CLK_SRC_XTAL`: 外部晶振时钟
  - **影响**: 影响I2C从设备的时钟精度和功耗

- **`send_buf_depth`** (类型: `uint32_t`)
  - **功能**: 设置内部发送环形缓冲区深度
  - **单位**: 字节
  - **作用**: 存储待发送给主设备的数据
  - **建议**: 根据应用需求设置，通常256-1024字节

- **`receive_buf_depth`** (类型: `uint32_t`)
  - **功能**: 设置接收内部软件缓冲区深度
  - **单位**: 字节
  - **作用**: 存储从主设备接收的数据
  - **建议**: 根据应用数据量设置，通常256-1024字节

- **`slave_addr`** (类型: `uint16_t`)
  - **功能**: 设置I2C从设备地址
  - **取值范围**: 7位地址：0x08-0x77，10位地址：0x000-0x3FF
  - **注意事项**: 避免使用保留地址（0x00-0x07, 0x78-0x7F）
  - **唯一性**: 同一总线上的设备地址必须唯一

- **`addr_bit_len`** (类型: `i2c_addr_bit_len_t`)
  - **功能**: 选择从设备的地址长度
  - **可选值**:
    - `I2C_ADDR_BIT_LEN_7`: 7位地址模式（标准模式）
    - `I2C_ADDR_BIT_LEN_10`: 10位地址模式（扩展模式）
  - **应用**: 大部分I2C设备使用7位地址

- **`intr_priority`** (类型: `int`)
  - **功能**: 设置I2C中断优先级
  - **取值范围**: 0表示使用默认优先级，其他值为1-7
  - **默认值**: 0（驱动会选择默认优先级）
  - **注意事项**: 优先级数值越小，优先级越高

- **`flags.allow_pd`** (类型: `uint32_t`, 1位)
  - **功能**: 允许在轻度睡眠时掉电
  - **取值**: 0-不允许，1-允许
  - **作用**: 启用后驱动会在睡眠前备份/恢复I2C寄存器
  - **权衡**: 可以节省功耗，但会消耗更多RAM

- **`flags.enable_internal_pullup`** (类型: `uint32_t`, 1位)
  - **功能**: 启用内部上拉电阻
  - **取值**: 0-禁用，1-启用
  - **注意事项**: 内部上拉电阻较弱，建议使用外部上拉
  - **应用场景**: 临时测试或低速通信时使用

- **`flags.broadcast_en`** (类型: `uint32_t`, 1位，部分芯片支持)
  - **功能**: 启用广播模式
  - **取值**: 0-禁用，1-启用
  - **作用**: 从设备可以响应广播地址(0x00)
  - **限制**: 不能与10位地址模式同时使用

### i2c_slave_event_callbacks_t

I2C从设备事件回调函数组结构体：

```c
typedef struct {
    i2c_slave_request_callback_t on_request;         // 主设备请求数据回调
    i2c_slave_received_callback_t on_receive;        // 从设备接收数据回调
} i2c_slave_event_callbacks_t;
```

**详细成员说明：**

- **`on_request`** (类型: `i2c_slave_request_callback_t`)
  - **功能**: 当主设备想要从从设备读取数据时触发的回调
  - **触发时机**: FIFO中没有数据而主设备发起读请求时
  - **处理**: 应在回调中通过`i2c_slave_write`向FIFO写入数据
  - **ISR上下文**: 在中断服务程序中运行，必须简短

- **`on_receive`** (类型: `i2c_slave_received_callback_t`)
  - **功能**: 从设备接收到完整数据包时触发的回调
  - **触发时机**: 从主设备接收到数据并完成一个事务后
  - **数据处理**: 可以在回调中处理接收到的数据
  - **ISR上下文**: 在中断服务程序中运行，必须简短

### i2c_slave_request_callback_t

```c
typedef bool (*i2c_slave_request_callback_t)(i2c_slave_dev_handle_t i2c_slave, 
                                              const i2c_slave_request_event_data_t *evt_data, 
                                              void *arg);
```

**详细说明：**

- **参数**: 
  - `i2c_slave`: I2C从设备句柄
  - `evt_data`: 请求事件数据（当前为空结构体）
  - `arg`: 用户自定义数据

- **返回值**: 
  - `true`: 唤醒了高优先级任务
  - `false`: 未唤醒高优先级任务

- **用途**: 主设备请求读取数据时，从设备应准备数据
- **实现**: 通常在回调中调用`i2c_slave_write`写入数据

### i2c_slave_received_callback_t

```c
typedef bool (*i2c_slave_received_callback_t)(i2c_slave_dev_handle_t i2c_slave, 
                                               const i2c_slave_rx_done_event_data_t *evt_data, 
                                               void *arg);
```

**详细说明：**

- **参数**: 
  - `i2c_slave`: I2C从设备句柄
  - `evt_data`: 接收完成事件数据，包含接收到的数据和长度
  - `arg`: 用户自定义数据

- **返回值**: 
  - `true`: 唤醒了高优先级任务
  - `false`: 未唤醒高优先级任务

- **用途**: 处理从主设备接收到的数据
- **数据访问**: 通过`evt_data->buffer`和`evt_data->length`访问数据

### i2c_slave_rx_done_event_data_t

从设备接收完成事件数据结构体：

```c
typedef struct {
    uint8_t *buffer;  // 接收缓冲区指针
    uint32_t length;  // 接收数据长度
} i2c_slave_rx_done_event_data_t;
```

**详细成员说明：**

- **`buffer`** (类型: `uint8_t *`)
  - **功能**: 指向接收到数据的缓冲区
  - **生命周期**: 仅在回调函数执行期间有效
  - **处理**: 需要在回调中立即处理或复制数据

- **`length`** (类型: `uint32_t`)
  - **功能**: 实际接收到的数据长度
  - **单位**: 字节
  - **用途**: 确定有效数据的范围

### i2c_slave_request_event_data_t

从设备请求事件数据结构体：

```c
typedef struct {
    // 当前为空结构体，预留扩展
} i2c_slave_request_event_data_t;
```

**详细说明：**

- **当前状态**: 空结构体，无具体成员
- **设计目的**: 为未来扩展预留空间
- **使用**: 当前版本中不包含具体数据

## 1. I2C Slave设备管理函数

### i2c_new_slave_device
**函数原型：**
```c
esp_err_t i2c_new_slave_device(const i2c_slave_config_t *slave_config, i2c_slave_dev_handle_t *ret_handle);
```
**参数：**
- `slave_config`：指向I2C从设备配置信息的结构体指针
- `ret_handle`：返回的I2C从设备句柄指针

**返回值：**
- `ESP_OK`：I2C从设备初始化成功
- `ESP_ERR_INVALID_ARG`：I2C设备初始化失败，参数无效
- `ESP_ERR_NO_MEM`：创建I2C设备失败，内存不足
- `ESP_ERR_NOT_SUPPORTED`：不支持的配置（如LP I2C从设备模式）

**作用：**初始化I2C从设备，配置GPIO引脚、时钟源、缓冲区、中断等参数

**详细功能：**
- **总线获取**: 获取或创建I2C总线句柄
- **引脚配置**: 设置SDA和SCL引脚为开漏模式
- **缓冲区创建**: 创建发送和接收环形缓冲区
- **中断安装**: 安装从设备中断处理程序
- **硬件初始化**: 配置从设备地址、FIFO阈值、时钟延展等
- **睡眠保持**: 可选的低功耗模式支持

### i2c_del_slave_device
**函数原型：**
```c
esp_err_t i2c_del_slave_device(i2c_slave_dev_handle_t i2c_slave);
```
**参数：**
- `i2c_slave`：要删除的I2C从设备句柄

**返回值：**
- `ESP_OK`：删除I2C设备成功
- `ESP_ERR_INVALID_ARG`：I2C设备初始化失败，参数无效

**作用：**取消初始化I2C从设备，释放所有相关资源

**详细功能：**
- **中断禁用**: 禁用所有从设备中断
- **引脚释放**: 恢复GPIO引脚到默认状态
- **总线释放**: 释放I2C总线句柄
- **缓冲区清理**: 删除环形缓冲区
- **内存释放**: 释放设备结构体内存

## 2. I2C Slave数据传输函数

### i2c_slave_write
**函数原型：**
```c
esp_err_t i2c_slave_write(i2c_slave_dev_handle_t i2c_slave, const uint8_t *data, uint32_t len, uint32_t *write_len, int timeout_ms);
```
**参数：**
- `i2c_slave`：I2C从设备句柄
- `data`：要写入的数据缓冲区
- `len`：要写入的数据长度
- `write_len`：实际写入的数据长度指针
- `timeout_ms`：超时时间（毫秒），-1表示永远等待

**返回值：**
- `ESP_OK`：I2C从设备写入成功
- `ESP_ERR_INVALID_ARG`：I2C从设备写入参数无效
- `ESP_ERR_TIMEOUT`：操作超时

**作用：**将数据写入硬件FIFO，如果写入长度大于硬件FIFO，则存储在软件缓冲区中

**详细机制：**
- **FIFO优先**: 优先填充硬件FIFO以获得最快响应
- **缓冲区备份**: 超出FIFO容量的数据存储在环形缓冲区
- **自动传输**: ISR会自动从缓冲区补充FIFO
- **非阻塞**: 可以在主设备请求前预先准备数据

**使用场景：**
- 响应主设备的读请求
- 预先准备待发送数据
- 状态或传感器数据上报

## 3. I2C Slave事件回调函数

### i2c_slave_register_event_callbacks
**函数原型：**
```c
esp_err_t i2c_slave_register_event_callbacks(i2c_slave_dev_handle_t i2c_slave, const i2c_slave_event_callbacks_t *cbs, void *user_data);
```
**参数：**

- `i2c_slave`：I2C从设备句柄
- `cbs`：回调函数组指针
- `user_data`：用户数据，将直接传递给回调函数

**返回值：**
- `ESP_OK`：设置I2C事务回调成功
- `ESP_ERR_INVALID_ARG`：设置I2C事务回调失败，参数无效
- `ESP_FAIL`：设置I2C事务回调失败，其他错误

**作用：**为I2C从设备注册事件回调函数，用于处理数据请求和接收事件

**注意事项：**
- **ISR上下文**: 回调函数在中断上下文中运行
- **IRAM要求**: 当启用CONFIG_I2C_ISR_IRAM_SAFE时，回调函数必须放在IRAM中
- **简短执行**: 回调函数必须快速执行，避免长时间阻塞
- **数据位置**: 用户数据必须在内部RAM中

## 4. 内部辅助函数（ISR函数）

### i2c_slave_isr_handler (中断服务程序)
**功能：**I2C从设备的主中断处理程序

### i2c_slave_handle_rx_fifo
**功能：**处理接收FIFO数据
**处理步骤：**
1. 从硬件FIFO读取数据
2. 将数据存储到接收环形缓冲区
3. 更新接收数据计数器

### i2c_slave_handle_tx_fifo
**功能：**处理发送FIFO数据
**处理步骤：**
1. 检查发送FIFO可用空间
2. 从发送环形缓冲区获取数据
3. 将数据写入硬件FIFO
4. 无数据时禁用发送中断

### i2c_slave_read_rx
**功能：**从接收缓冲区读取数据到用户缓冲区
**特性：**
- 非阻塞读取
- 支持部分读取
- 自动管理环形缓冲区项目

### i2c_slave_handle_stretch_event
**函数原型：**
```c
IRAM_ATTR static bool i2c_slave_handle_stretch_event(i2c_slave_dev_t *i2c_slave, uint32_t rx_fifo_exist_len);
```
**参数：**
- `i2c_slave`：I2C从设备结构体指针
- `rx_fifo_exist_len`：接收FIFO中现有数据长度

**返回值：**
- `true`：唤醒了高优先级任务
- `false`：未唤醒高优先级任务

**作用：**处理I2C从设备时钟延展事件（仅支持时钟延展检测的芯片）

### i2c_slave_isr_handler
**函数原型：**
```c
IRAM_ATTR static void i2c_slave_isr_handler(void *arg);
```
**参数：**
- `arg`：传递给ISR的参数，通常是I2C从设备结构体指针

**作用：**I2C从设备的主中断服务程序，处理所有I2C从设备相关中断

**详细功能：**
- **中断状态获取**: 读取并清除中断状态标志
- **FIFO状态检查**: 获取接收FIFO中的数据长度
- **读写状态检测**: 判断当前是主设备读取还是写入操作
- **多中断处理**: 同时处理多个中断源
- **任务唤醒管理**: 统一管理高优先级任务的唤醒

**中断处理顺序：**
1. 获取中断掩码和FIFO状态
2. 处理接收FIFO水位中断
3. 处理从设备完成中断
4. 处理时钟延展中断（如支持）
5. 处理发送FIFO水位中断
6. 统一处理任务唤醒

**特殊处理：**
- **ESP32-C5工作区**: 包含针对C5芯片数字Bug的特殊处理代码
- **自动启动**: 避免次级事务的潜在问题

### i2c_slave_device_destroy
**函数原型：**
```c
static esp_err_t i2c_slave_device_destroy(i2c_slave_dev_handle_t i2c_slave);
```
**参数：**
- `i2c_slave`：要销毁的I2C从设备句柄

**返回值：**
- `ESP_OK`：设备销毁成功
- 其他错误码：销毁过程中发生错误

**作用：**销毁I2C从设备，释放所有相关资源

**详细功能：**
- **中断禁用**: 禁用所有从设备相关中断
- **引脚清理**: 调用通用引脚去初始化函数
- **总线释放**: 释放I2C总线句柄
- **缓冲区清理**: 删除接收和发送环形缓冲区
- **信号量清理**: 删除操作互斥信号量
- **内存释放**: 释放接收缓冲区和设备结构体内存

**清理顺序：**
1. 禁用硬件中断
2. 清理GPIO引脚配置
3. 释放总线资源
4. 删除软件缓冲区
5. 清理同步原语
6. 释放内存资源

**错误处理：**
- 即使某些清理步骤失败，仍继续执行后续清理
- 返回第一个遇到的错误码
- 确保内存不泄露
