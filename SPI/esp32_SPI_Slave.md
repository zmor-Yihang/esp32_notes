# ESP32 SPI Slave API 完整函数参考

## 0. SPI Slave主要数据结构

### spi_slave_interface_config_t

SPI从设备接口配置结构体，用于配置SPI从设备模式：

```c
typedef struct {
    int spics_io_num;               ///< CS GPIO引脚编号
    uint32_t flags;                 ///< 位掩码标志，SPI_SLAVE_*常量的或运算
    int queue_size;                 ///< 事务队列大小，决定可以同时"在空中"的事务数量
    uint8_t mode;                   ///< SPI模式，表示(CPOL, CPHA)配置对：
                                    ///< 0: (0, 0), 1: (0, 1), 2: (1, 0), 3: (1, 1)
    slave_transaction_cb_t post_setup_cb;  ///< SPI寄存器加载新数据后调用的回调
    slave_transaction_cb_t post_trans_cb;  ///< 事务完成后调用的回调
} spi_slave_interface_config_t;
```

**详细成员说明：**

- **`spics_io_num`** (类型: `int`)
  - **功能**: CS(片选)信号的GPIO引脚编号
  - **取值**: 有效GPIO编号
  - **说明**: 从设备通过CS信号知道主机何时与其通信

- **`flags`** (类型: `uint32_t`)
  - **功能**: 从设备特性标志
  - **可用标志**:
    - `SPI_SLAVE_TXBIT_LSBFIRST`: 发送命令/地址/数据LSB优先
    - `SPI_SLAVE_RXBIT_LSBFIRST`: 接收数据LSB优先
    - `SPI_SLAVE_BIT_LSBFIRST`: 收发都LSB优先
    - `SPI_SLAVE_NO_RETURN_RESULT`: 完成时不返回描述符(使用post_trans_cb通知)
  - **组合**: 可以通过或运算组合多个标志

- **`queue_size`** (类型: `int`)
  - **功能**: 事务队列大小
  - **说明**: 设置可以同时排队(已通过spi_slave_queue_trans排队但尚未通过spi_slave_get_trans_result完成)的事务数量
  - **建议**: 根据应用需求设置，通常3-10个

- **`mode`** (类型: `uint8_t`)
  - **功能**: SPI时钟模式
  - **取值**: 
    - 0: CPOL=0, CPHA=0 (时钟空闲时为低，第一个边沿采样)
    - 1: CPOL=0, CPHA=1 (时钟空闲时为低，第二个边沿采样)
    - 2: CPOL=1, CPHA=0 (时钟空闲时为高，第一个边沿采样)
    - 3: CPOL=1, CPHA=1 (时钟空闲时为高，第二个边沿采样)
  - **说明**: 必须与主机模式匹配

- **`post_setup_cb`** (类型: `slave_transaction_cb_t`)
  - **功能**: SPI寄存器加载新数据后调用的回调函数
  - **时机**: 事务开始前，硬件准备就绪时
  - **上下文**: 在ISR中调用
  - **性能**: 应放在IRAM中以获得最佳性能
  - **用途**: 准备下一次传输的数据

- **`post_trans_cb`** (类型: `slave_transaction_cb_t`)
  - **功能**: 事务完成后调用的回调函数
  - **时机**: 主机完成一次传输后
  - **上下文**: 在ISR中调用
  - **性能**: 应放在IRAM中以获得最佳性能
  - **用途**: 处理接收到的数据或通知应用层

### spi_slave_transaction_t

SPI从设备事务描述符结构体，描述一次从设备端的SPI传输：

```c
struct spi_slave_transaction_t {
    uint32_t flags;                 ///< 位掩码标志，SPI_SLAVE_TRANS_*常量
    size_t length;                  ///< 总数据长度(位)
    size_t trans_len;               ///< 实际传输的数据长度(位)，由驱动填充
    const void *tx_buffer;          ///< 指向发送缓冲区的指针，NULL表示无MOSI阶段
    void *rx_buffer;                ///< 指向接收缓冲区的指针，NULL表示无MISO阶段
    void *user;                     ///< 用户定义变量，可用于存储事务ID等
};
```

**详细成员说明：**

- **`flags`** (类型: `uint32_t`)
  - **功能**: 事务特性标志
  - **可用标志**:
    - `SPI_SLAVE_TRANS_DMA_BUFFER_ALIGN_AUTO`: 如果用户缓冲区不满足硬件对齐或DMA能力要求，自动重新分配DMA缓冲区
  - **说明**: 自动对齐可能会损失一些内存和性能

- **`length`** (类型: `size_t`)
  - **功能**: 总数据长度
  - **单位**: 位(bits)
  - **说明**: 可以不是字节对齐的
  - **限制**: 不能超过初始化时设置的max_transfer_sz

- **`trans_len`** (类型: `size_t`)
  - **功能**: 实际传输的数据长度
  - **单位**: 位(bits)
  - **说明**: 由驱动程序在传输完成后填充
  - **用途**: 获知主机实际发送/接收了多少数据

- **`tx_buffer`** (类型: `const void*`)
  - **功能**: 指向发送缓冲区的指针
  - **要求**: 
    - 使用DMA时必须在DMA可访问内存中
    - 必须字对齐(地址%4==0)
    - 长度应为4字节的倍数
  - **NULL值**: 表示无发送数据(仅接收)

- **`rx_buffer`** (类型: `void*`)
  - **功能**: 指向接收缓冲区的指针
  - **要求**: 
    - 使用DMA时必须在DMA可访问内存中
    - 必须字对齐(地址%4==0)
    - 长度应为4字节的倍数
  - **NULL值**: 表示无接收数据(仅发送)
  - **注意**: ESP32上如果长度不是字对齐，最后一个字的内存仍会被DMA硬件覆盖

- **`user`** (类型: `void*`)
  - **功能**: 用户定义变量
  - **用途**: 可用于存储事务ID或其他上下文信息
  - **说明**: 驱动不会修改此值

### slave_transaction_cb_t

从设备事务回调函数类型定义：

```c
typedef void(*slave_transaction_cb_t)(spi_slave_transaction_t *trans);
```

**参数：**
- trans: 事务描述符指针

**说明：**
- 在ISR上下文中调用
- 必须快速执行，不能阻塞
- 不能调用FreeRTOS阻塞API
- 应放在IRAM中以获得最佳性能

---

## 1. SPI Slave初始化和释放函数

### spi_slave_initialize
**函数原型：**
```c
esp_err_t spi_slave_initialize(spi_host_device_t host, const spi_bus_config_t *bus_config, const spi_slave_interface_config_t *slave_config, spi_dma_chan_t dma_chan);
```
**参数：**
host：用作SPI从设备接口的SPI外设(SPI2_HOST或SPI3_HOST)
bus_config：指向spi_bus_config_t结构体的指针，指定主机应如何初始化
slave_config：指向spi_slave_interface_config_t结构体的指针，指定从设备接口的详细信息
dma_chan：DMA通道选择
  - SPI_DMA_DISABLED：禁用DMA，限制事务大小
  - SPI_DMA_CH_AUTO：让驱动自动分配DMA通道(推荐)
  - SPI_DMA_CH1/CH2：手动选择DMA通道(仅ESP32)
**返回值：**
ESP_OK：初始化成功
ESP_ERR_INVALID_ARG：配置参数无效
ESP_ERR_INVALID_STATE：主机已在使用中
ESP_ERR_NOT_FOUND：没有可用的DMA通道
ESP_ERR_NO_MEM：内存不足
**作用：**将SPI总线初始化为从设备接口并默认启用。如果选择DMA通道，任何发送和接收缓冲区都应分配在DMA能访问的内存中

**重要说明：**
- SPI0/1不支持从设备模式
- SPI的ISR总是在调用此函数的核心上执行
- 不要在该核心上使ISR饿死，否则SPI事务将无法处理
- 初始化后从设备默认是启用状态

### spi_slave_free
**函数原型：**
```c
esp_err_t spi_slave_free(spi_host_device_t host);
```
**参数：**
host：要释放的SPI外设
**返回值：**
ESP_OK：释放成功
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_INVALID_STATE：总线上的设备未全部释放
**作用：**释放声明为SPI从设备接口的SPI总线

**注意事项：**
- 释放前确保所有事务已完成
- 释放后GPIO引脚会被重置
- DMA资源会被释放

---

## 2. SPI Slave使能控制函数

### spi_slave_enable
**函数原型：**
```c
esp_err_t spi_slave_enable(spi_host_device_t host);
```
**参数：**
host：要启用的SPI外设
**返回值：**
ESP_OK：成功启用
ESP_ERR_INVALID_ARG：不支持的主机
ESP_ERR_INVALID_STATE：外设已经启用
**作用：**为已初始化的SPI主机启用从设备功能。在spi_slave_initialize后不需要额外调用此函数，因为初始化时已经启用

**使用场景：**
- 在调用spi_slave_disable后重新启用
- 动态控制从设备的启用状态

### spi_slave_disable
**函数原型：**
```c
esp_err_t spi_slave_disable(spi_host_device_t host);
```
**参数：**
host：要禁用的SPI外设
**返回值：**
ESP_OK：成功禁用
ESP_ERR_INVALID_ARG：不支持的主机
ESP_ERR_INVALID_STATE：外设已经禁用
**作用：**为已初始化的SPI主机禁用从设备功能

**使用场景：**
- 暂时停止从设备功能
- 节省功耗
- 需要重新配置前先禁用

---

## 3. SPI Slave事务管理函数

### spi_slave_queue_trans
**函数原型：**
```c
esp_err_t spi_slave_queue_trans(spi_host_device_t host, const spi_slave_transaction_t *trans_desc, TickType_t ticks_to_wait);
```
**参数：**
host：作为从设备的SPI外设
trans_desc：要执行的事务描述符。非const因为可能需要写回状态到事务描述中
ticks_to_wait：等待队列有空间的超时时间，使用portMAX_DELAY表示永不超时
**返回值：**
ESP_OK：成功排队事务
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_NO_MEM：设置了SPI_SLAVE_TRANS_DMA_BUFFER_ALIGN_AUTO标志但没有空闲内存
ESP_ERR_INVALID_STATE：缓存和内存之间的数据同步失败
**作用：**将SPI事务排队等待执行。此函数将trans_desc中缓冲区的所有权交给SPI从设备驱动程序；应用程序不应访问此内存，直到调用spi_slave_get_trans_result将所有权交还给应用程序

**重要说明：**
- 在ESP32上，如果传输长度不是字对齐，rx缓冲区最后一个字的内存仍会被DMA硬件覆盖
- 此函数可能会阻塞，取决于队列是否已满
- 没有直接启动SPI操作，下一个排队的事务将在主机拉低CS并发送时钟信号时发生
- 事务队列大小在初始化时通过queue_size指定

### spi_slave_get_trans_result
**函数原型：**
```c
esp_err_t spi_slave_get_trans_result(spi_host_device_t host, spi_slave_transaction_t **trans_desc, TickType_t ticks_to_wait);
```
**参数：**
host：作为从设备的SPI外设
trans_desc：指向能够包含已执行事务描述符指针的变量的指针
ticks_to_wait：等待返回项的超时时间，使用portMAX_DELAY表示永不超时
**返回值：**
ESP_OK：成功获取结果
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_NOT_SUPPORTED：设置了SPI_SLAVE_NO_RETURN_RESULT标志
**作用：**获取之前排队的SPI事务的结果。此例程会等待直到给定设备的事务成功完成，然后返回已完成事务的描述符

**重要说明：**
- 必须最终为通过spi_slave_queue_trans排队的任何事务使用此函数
- 此函数会阻塞直到事务完成
- 如果设置了SPI_SLAVE_NO_RETURN_RESULT标志，则不支持此API

### spi_slave_transmit
**函数原型：**
```c
esp_err_t spi_slave_transmit(spi_host_device_t host, spi_slave_transaction_t *trans_desc, TickType_t ticks_to_wait);
```
**参数：**
host：作为从设备的SPI外设
trans_desc：指向能够包含已执行事务描述符指针的变量的指针。非const因为可能需要写回状态
ticks_to_wait：等待返回项的超时时间，使用portMAX_DELAY表示永不超时
**返回值：**
ESP_OK：成功完成传输
ESP_ERR_INVALID_ARG：参数无效
**作用：**执行SPI事务。本质上与先调用spi_slave_queue_trans再调用spi_slave_get_trans_result相同。当还有未使用spi_slave_get_trans_result完成的排队事务时，不要使用此函数

**注意事项：**
- 这是阻塞函数
- 简化的API，适合简单的同步传输
- 内部调用queue_trans和get_trans_result

---

## 4. SPI Slave ISR函数

### spi_slave_queue_trans_isr
**函数原型：**
```c
esp_err_t spi_slave_queue_trans_isr(spi_host_device_t host, const spi_slave_transaction_t *trans_desc);
```
**参数：**
host：作为从设备的SPI外设
trans_desc：要执行的事务描述符
**返回值：**
ESP_OK：成功排队事务
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_NO_MEM：队列已满
**作用：**在ISR上下文中将SPI事务排队等待执行

**重要说明：**
- 此函数只能在ISR中调用
- 不支持自动DMA缓冲区对齐
- 缓冲区必须满足DMA要求
- 适用于高实时性要求的场景

### spi_slave_queue_reset
**函数原型：**
```c
esp_err_t spi_slave_queue_reset(spi_host_device_t host);
```
**参数：**
host：作为从设备的SPI外设
**返回值：**
ESP_OK：成功重置队列
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_INVALID_STATE：无法重置队列
**作用：**重置SPI从设备事务队列。调用此函数后，SPI从设备事务队列将被重置。已挂载在硬件上的事务不会被重置，可以被下一个trans_queue覆盖

**重要说明：**
- 不应在相应的SPI主机正在进行SPI事务时调用此函数
- 硬件上的事务不受影响
- 队列中的待处理事务将被清除

### spi_slave_queue_reset_isr
**函数原型：**
```c
esp_err_t spi_slave_queue_reset_isr(spi_host_device_t host);
```
**参数：**
host：作为从设备的SPI外设
**返回值：**
ESP_OK：成功重置队列
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_INVALID_STATE：无法重置队列
**作用：**在ISR上下文中重置SPI从设备事务队列

**使用场景：**
- 在中断处理程序中快速重置队列
- 处理错误情况时清空队列

---

## 常用宏定义

### SPI从设备标志
```c
#define SPI_SLAVE_TXBIT_LSBFIRST          (1<<0)  ///< 发送命令/地址/数据LSB优先
#define SPI_SLAVE_RXBIT_LSBFIRST          (1<<1)  ///< 接收数据LSB优先
#define SPI_SLAVE_BIT_LSBFIRST            (SPI_SLAVE_TXBIT_LSBFIRST|SPI_SLAVE_RXBIT_LSBFIRST)
#define SPI_SLAVE_NO_RETURN_RESULT        (1<<2)  ///< 完成时不返回描述符(使用post_trans_cb通知)
```

### SPI从设备事务标志
```c
#define SPI_SLAVE_TRANS_DMA_BUFFER_ALIGN_AUTO   (1<<0)  ///< 自动重新分配DMA缓冲区
```
