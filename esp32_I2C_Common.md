# ESP32 I2C Common API 完整函数参考

## 0. I2C Common主要数据结构

### i2c_platform_t

I2C平台管理结构体，用于管理所有I2C总线实例的全局状态：

```c
typedef struct i2c_platform_t {
    _lock_t mutex;                        // 平台级互斥锁
    i2c_bus_handle_t buses[SOC_I2C_NUM];  // I2C总线实例数组
    uint32_t count[SOC_I2C_NUM];          // 引用计数，保护总线安装/卸载
} i2c_platform_t;
```

**详细成员说明：**

- **`mutex`** (类型: `_lock_t`)
  - **功能**: 平台级互斥锁，保护对总线数组的并发访问
  - **作用**: 确保多线程环境下总线分配和释放的原子性
  - **使用**: 在获取和释放总线时自动使用

- **`buses`** (类型: `i2c_bus_handle_t[SOC_I2C_NUM]`)
  - **功能**: 存储所有I2C总线实例的句柄数组
  - **大小**: `SOC_I2C_NUM` 定义了芯片支持的I2C总线数量
  - **管理**: NULL表示该端口未使用，非NULL表示已分配

- **`count`** (类型: `uint32_t[SOC_I2C_NUM]`)
  - **功能**: 每个端口的引用计数器
  - **作用**: 跟踪每个总线被引用的次数，支持多设备共享同一总线
  - **生命周期**: 计数为0时才真正释放总线资源

### i2c_master_status_t

I2C主设备FSM（有限状态机）状态枚举：

```c
typedef enum {
    I2C_STATUS_READ,      // 当前主命令的读状态
    I2C_STATUS_WRITE,     // 当前主命令的写状态
    I2C_STATUS_START,     // 当前主命令的启动状态
    I2C_STATUS_STOP,      // 当前主命令的停止状态
    I2C_STATUS_IDLE,      // 当前主命令的空闲状态
    I2C_STATUS_ACK_ERROR, // 当前主命令的ACK错误状态
    I2C_STATUS_DONE,      // I2C命令完成
    I2C_STATUS_TIMEOUT,   // I2C总线状态错误，操作超时
} i2c_master_status_t;
```

**详细状态说明：**

- **`I2C_STATUS_READ`**
  - **含义**: 主设备正在执行读操作
  - **触发**: 当I2C硬件正在从从设备读取数据时
  - **后续**: 可能转为DONE、TIMEOUT或ACK_ERROR状态

- **`I2C_STATUS_WRITE`**
  - **含义**: 主设备正在执行写操作
  - **触发**: 当I2C硬件正在向从设备写入数据时
  - **后续**: 根据ACK响应转为相应状态

- **`I2C_STATUS_START`**
  - **含义**: 主设备发送START或RESTART条件
  - **作用**: 表示一个新的I2C事务开始
  - **时机**: 每个I2C事务的开始阶段

- **`I2C_STATUS_STOP`**
  - **含义**: 主设备发送STOP条件
  - **作用**: 表示当前I2C事务结束
  - **释放**: 释放I2C总线供其他主设备使用

- **`I2C_STATUS_IDLE`**
  - **含义**: I2C总线处于空闲状态
  - **特征**: 没有进行中的事务
  - **初始状态**: 总线初始化后的默认状态

- **`I2C_STATUS_ACK_ERROR`**
  - **含义**: 从设备未响应ACK信号
  - **原因**: 设备地址错误、设备忙碌或硬件连接问题
  - **处理**: 通常需要重试或检查硬件连接

- **`I2C_STATUS_DONE`**
  - **含义**: I2C命令序列成功完成
  - **结果**: 所有预期的数据传输都已完成
  - **后续**: 可以进行下一个事务

- **`I2C_STATUS_TIMEOUT`**
  - **含义**: I2C操作超时
  - **原因**: 总线被拉低、时钟延展过长或硬件故障
  - **恢复**: 可能需要总线重置

### i2c_master_event_t

I2C主设备事件类型枚举：

```c
typedef enum {
    I2C_EVENT_ALIVE,      // I2C总线处于活跃状态
    I2C_EVENT_DONE,       // I2C总线事务完成
    I2C_EVENT_NACK,       // I2C总线NACK响应
    I2C_EVENT_TIMEOUT,    // I2C总线超时
} i2c_master_event_t;
```

**详细事件说明：**

- **`I2C_EVENT_ALIVE`**
  - **含义**: I2C总线正常工作状态
  - **用途**: 表示总线可以接受新的事务
  - **监控**: 用于健康检查和状态监控

- **`I2C_EVENT_DONE`**
  - **含义**: I2C事务成功完成
  - **触发**: 所有命令执行完毕且无错误
  - **回调**: 异步操作的完成通知

- **`I2C_EVENT_NACK`**
  - **含义**: 接收到NACK（非确认）信号
  - **处理**: 错误事件，需要适当的错误处理
  - **调试**: 有助于诊断通信问题

- **`I2C_EVENT_TIMEOUT`**
  - **含义**: 操作超时事件
  - **影响**: 事务被强制终止
  - **恢复**: 可能需要总线重置和重试

### i2c_master_command_t

I2C主设备命令类型枚举：

```c
typedef enum {
    I2C_MASTER_CMD_START,    // 启动或重启条件
    I2C_MASTER_CMD_WRITE,    // 写操作
    I2C_MASTER_CMD_READ,     // 读操作
    I2C_MASTER_CMD_STOP,     // 停止条件
} i2c_master_command_t;
```

**详细命令说明：**

- **`I2C_MASTER_CMD_START`**
  - **功能**: 发送START或RESTART条件
  - **时机**: 每个I2C事务的开始
  - **协议**: 拉低SDA线（SCL为高）

- **`I2C_MASTER_CMD_WRITE`**
  - **功能**: 向从设备写入数据
  - **包含**: 设备地址、寄存器地址或实际数据
  - **确认**: 需要等待从设备的ACK响应

- **`I2C_MASTER_CMD_READ`**
  - **功能**: 从从设备读取数据
  - **确认**: 主设备发送ACK（继续读取）或NACK（结束读取）
  - **缓冲**: 数据存储到指定缓冲区

- **`I2C_MASTER_CMD_STOP`**
  - **功能**: 发送STOP条件
  - **时机**: I2C事务结束
  - **协议**: 拉高SDA线（SCL为高）

### i2c_ack_value_t

I2C ACK值枚举：

```c
typedef enum {
    I2C_ACK_VAL = 0,  // 确认(ACK)信号
    I2C_NACK_VAL = 1, // 非确认(NACK)信号
} __attribute__((packed)) i2c_ack_value_t;
```

**详细值说明：**

- **`I2C_ACK_VAL`**
  - **含义**: 发送确认信号
  - **用途**: 表示成功接收数据，可以继续传输
  - **时机**: 读取多字节数据的中间阶段

- **`I2C_NACK_VAL`**
  - **含义**: 发送非确认信号
  - **用途**: 表示传输结束，不再接收数据
  - **时机**: 读取操作的最后一个字节后

### i2c_master_callback_t

```c
typedef bool (*i2c_master_callback_t)(i2c_master_dev_handle_t i2c_dev, 
                                       const i2c_master_event_data_t *evt_data, 
                                       void *arg);
```

**详细说明：**

- **参数**: 
  - `i2c_dev`: I2C设备句柄
  - `evt_data`: 事件数据，包含事件类型等信息
  - `arg`: 用户自定义数据

- **返回值**: 
  - `true`: 唤醒了高优先级任务
  - `false`: 未唤醒高优先级任务

- **上下文**: 在ISR（中断服务程序）上下文中运行
- **限制**: 函数必须简短，避免阻塞操作

### i2c_slave_received_callback_t

```c
typedef bool (*i2c_slave_received_callback_t)(i2c_slave_dev_handle_t i2c_slave, 
                                               const i2c_slave_rx_done_event_data_t *evt_data, 
                                               void *arg);
```

**详细说明：**

- **用途**: I2C从设备接收数据完成的回调
- **触发**: 从设备接收到完整的数据包
- **处理**: 可以在回调中处理接收到的数据

## 1. I2C总线生命周期管理函数

### i2c_acquire_bus_handle
**函数原型：**
```c
esp_err_t i2c_acquire_bus_handle(i2c_port_num_t port_num, i2c_bus_handle_t *i2c_new_bus, i2c_bus_mode_t mode);
```
**参数：**
- `port_num`：I2C端口号，-1表示自动选择可用端口
- `i2c_new_bus`：返回的I2C总线句柄指针
- `mode`：总线模式（主设备或从设备模式）

**返回值：**
- `ESP_OK`：获取总线句柄成功
- `ESP_ERR_INVALID_STATE`：总线已被占用
- `ESP_ERR_NOT_FOUND`：没有可用的空闲总线
- `ESP_ERR_NO_MEM`：内存不足

**作用：**获取指定或自动选择的I2C总线句柄，初始化总线硬件和相关资源

**详细说明：**
- 支持自动端口选择：当`port_num`为-1时，自动寻找可用端口
- 引用计数管理：同一总线可被多个设备共享使用
- 硬件初始化：包括时钟使能、寄存器重置、HAL层初始化
- 睡眠保持：支持低功耗模式下的寄存器状态保持

### i2c_release_bus_handle
**函数原型：**
```c
esp_err_t i2c_release_bus_handle(i2c_bus_handle_t i2c_bus);
```
**参数：**
- `i2c_bus`：要释放的I2C总线句柄

**返回值：**
- `ESP_OK`：释放总线句柄成功
- `ESP_ERR_INVALID_STATE`：总线未完全释放

**作用：**释放I2C总线句柄，减少引用计数，当计数归零时释放所有资源

**详细说明：**
- 引用计数管理：只有当引用计数为0时才真正释放资源
- 资源清理：包括中断处理、电源管理锁、时钟源等
- 硬件关闭：禁用I2C模块时钟，重置硬件状态
- 内存释放：释放总线结构体内存

### i2c_bus_occupied
**函数原型：**
```c
bool i2c_bus_occupied(i2c_port_num_t port_num);
```
**参数：**
- `port_num`：要检查的I2C端口号

**返回值：**
- `true`：总线已被占用
- `false`：总线可用

**作用：**检查指定I2C端口是否已被占用，用于端口选择和状态查询

**应用场景：**
- 端口可用性检查
- 自动端口选择的辅助函数
- 系统状态监控

## 2. 时钟源管理函数

### i2c_select_periph_clock
**函数原型：**
```c
esp_err_t i2c_select_periph_clock(i2c_bus_handle_t handle, soc_module_clk_t clk_src);
```
**参数：**
- `handle`：I2C总线句柄
- `clk_src`：要选择的时钟源

**返回值：**
- `ESP_OK`：时钟选择成功
- `ESP_ERR_INVALID_ARG`：总线句柄为空
- `ESP_ERR_INVALID_STATE`：时钟源冲突

**作用：**为I2C总线选择和配置外设时钟源，设置时钟频率

**支持的时钟源：**
- `I2C_CLK_SRC_APB`：APB时钟（常用）
- `I2C_CLK_SRC_XTAL`：外部晶振时钟
- `I2C_CLK_SRC_RC_FAST`：内部RC快速时钟（低功耗）

**详细功能：**
- 时钟冲突检测：防止同一总线使用不同时钟源
- 频率获取：自动获取时钟源实际频率
- 电源管理：创建相应的PM锁以保持时钟稳定
- 低功耗支持：RC_FAST时钟的特殊处理

## 3. GPIO引脚配置函数

### i2c_common_set_pins
**函数原型：**
```c
esp_err_t i2c_common_set_pins(i2c_bus_handle_t handle);
```
**参数：**
- `handle`：I2C总线句柄，包含SDA和SCL引脚配置信息

**返回值：**
- `ESP_OK`：引脚配置成功
- `ESP_ERR_INVALID_ARG`：引脚参数无效

**作用：**配置I2C总线的SDA和SCL引脚，包括GPIO模式、上拉电阻、信号路由

**配置内容：**

#### 高性能I2C (HP I2C)引脚配置：
- **GPIO模式设置**：
  - 开漏输出模式（Open-drain）
  - 输入使能
  - 初始电平设置为高

- **上拉电阻配置**：
  - 根据配置启用或禁用内部上拉
  - 禁用下拉电阻

- **信号路由**：
  - 通过GPIO矩阵连接I2C信号
  - 配置输入和输出信号路径

#### 低功耗I2C (LP I2C)引脚配置：
- **RTC GPIO配置**：
  - 使用RTC域的GPIO功能
  - 开漏输入输出模式
  - 上拉/下拉配置

- **信号连接**：
  - IO MUX直连（某些芯片）
  - LP GPIO矩阵路由（支持的芯片）

### i2c_common_deinit_pins
**函数原型：**
```c
esp_err_t i2c_common_deinit_pins(i2c_bus_handle_t handle);
```
**参数：**
- `handle`：I2C总线句柄

**返回值：**
- `ESP_OK`：引脚去初始化成功

**作用：**取消I2C引脚配置，恢复引脚到默认状态

**处理内容：**

#### 高性能I2C引脚去初始化：
- 禁用GPIO输出
- 断开GPIO矩阵信号连接
- 将输入信号连接到常量零

#### 低功耗I2C引脚去初始化：
- 取消RTC GPIO初始化
- 断开LP GPIO矩阵连接（如支持）
- 恢复引脚默认状态

## 4. 睡眠保持功能（仅限HP I2C）

### i2c_create_retention_module
**函数原型：**
```c
void i2c_create_retention_module(i2c_bus_handle_t handle);
```
**参数：**
- `handle`：I2C总线句柄

**作用：**创建睡眠保持模块，在轻度睡眠期间保持I2C寄存器状态

**功能特性：**
- **寄存器备份**：睡眠前自动备份I2C寄存器
- **状态恢复**：唤醒后自动恢复I2C配置
- **电源域关闭**：允许I2C电源域在睡眠时关闭
- **功耗优化**：以RAM消耗换取更低的睡眠功耗

**限制条件：**
- 仅支持高性能I2C（HP I2C）
- 需要CONFIG_PM_ENABLE开启
- LP I2C不支持此功能
