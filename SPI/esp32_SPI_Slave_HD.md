# ESP32 SPI Slave HD API 完整函数参考

## 0. SPI Slave HD主要数据结构

### spi_slave_hd_data_t

SPI Slave HD数据描述符，用于描述要发送或接收的数据：

```c
typedef struct {
    uint8_t* data;                              ///< 缓冲区指针，必须是DMA可访问的
    size_t   len;                               ///< 要发送/接收的数据长度
    size_t   trans_len;                         ///< 实际传输的数据长度
    uint32_t flags;                             ///< 标志位
    void*    arg;                               ///< 用户自定义参数
} spi_slave_hd_data_t;
```

**详细成员说明：**

- **`data`** (类型: `uint8_t*`)
  - **功能**: 指向要发送或接收的数据缓冲区
  - **要求**: 必须是DMA可访问的内存（使用`heap_caps_malloc(size, MALLOC_CAP_DMA)`分配）
  - **接收方向**: 缓冲区长度应为4字节的倍数，否则多余部分会被截断
  - **注意事项**: 不能使用栈上的局部变量或常量区的数据

- **`len`** (类型: `size_t`)
  - **功能**: 要发送或接收的数据长度（字节）
  - **发送方向**: 指定要发送的数据长度
  - **接收方向**: 指定缓冲区的最大容量，应为4字节的倍数
  - **限制**: 不能超过驱动配置的最大传输长度

- **`trans_len`** (类型: `size_t`)
  - **功能**: 实际传输的数据长度（字节）
  - **接收方向**: 由驱动填充，表示主机实际发送的数据量
  - **发送方向**: 无意义，可以忽略
  - **用途**: 用于确定接收到的有效数据长度

- **`flags`** (类型: `uint32_t`)
  - **功能**: 事务标志位
  - **可用标志**:
    - `SPI_SLAVE_HD_TRANS_DMA_BUFFER_ALIGN_AUTO`: 自动重新分配DMA对齐缓冲区
  - **说明**: 位掩码，可以使用按位OR组合多个标志

- **`arg`** (类型: `void*`)
  - **功能**: 用户自定义的参数指针
  - **用途**: 用于在回调函数中识别特定的事务
  - **应用**: 可以指向用户的上下文数据结构

### spi_slave_hd_event_t

SPI Slave HD事件信息结构体，在回调函数中传递事件信息：

```c
typedef struct {
    spi_event_t          event;                 ///< 事件类型
    spi_slave_hd_data_t* trans;                 ///< 相关的事务描述符
} spi_slave_hd_event_t;
```

**详细成员说明：**

- **`event`** (类型: `spi_event_t`)
  - **功能**: 指示发生的事件类型
  - **可能的值**:
    - `SPI_EV_BUF_TX`: 主机从共享缓冲区读取
    - `SPI_EV_BUF_RX`: 主机向共享缓冲区写入
    - `SPI_EV_SEND`: DMA发送完成
    - `SPI_EV_RECV`: DMA接收完成
    - `SPI_EV_CMD9`: 接收到CMD9命令
    - `SPI_EV_CMDA`: 接收到CMDA命令

- **`trans`** (类型: `spi_slave_hd_data_t*`)
  - **功能**: 指向相关的事务描述符
  - **有效事件**: 仅对`SPI_EV_SEND`和`SPI_EV_RECV`事件有效
  - **其他事件**: 为NULL
  - **用途**: 获取事务的详细信息（数据指针、长度等）

### slave_cb_t

SPI Slave HD回调函数类型定义：

```c
typedef bool (*slave_cb_t)(void* arg, spi_slave_hd_event_t* event, BaseType_t* awoken);
```

**参数说明：**

- **`arg`** (类型: `void*`)
  - **说明**: 用户自定义的参数，在配置时指定
  - **用途**: 通常指向SPI Slave HD实例的上下文

- **`event`** (类型: `spi_slave_hd_event_t*`)
  - **说明**: 指向事件信息的指针
  - **内容**: 包含事件类型和相关的事务描述符

- **`awoken`** (类型: `BaseType_t*`)
  - **说明**: FreeRTOS上下文切换标志
  - **用途**: 如果回调解锁了更高优先级的任务，应设置为pdTRUE

**返回值：**
- `true`: 将事务加入返回队列
- `false`: 不加入返回队列，用户将负责管理事务

**注意事项：**
- 回调函数在ISR上下文中运行
- 必须快速执行，不能阻塞
- 不能调用FreeRTOS的阻塞API（使用FromISR版本）

### spi_slave_chan_t

SPI Slave HD数据通道枚举：

```c
typedef enum {
    SPI_SLAVE_CHAN_TX = 0,                      ///< 输出通道（RDDMA）
    SPI_SLAVE_CHAN_RX = 1,                      ///< 输入通道（WRDMA）
} spi_slave_chan_t;
```

**详细说明：**

- **`SPI_SLAVE_CHAN_TX`**
  - **功能**: 发送通道（从从机到主机）
  - **方向**: 从机发送，主机读取
  - **DMA**: 使用RDDMA（读DMA）

- **`SPI_SLAVE_CHAN_RX`**
  - **功能**: 接收通道（从主机到从机）
  - **方向**: 主机发送，从机接收
  - **DMA**: 使用WRDMA（写DMA）

### spi_slave_hd_callback_config_t

SPI Slave HD回调配置结构体：

```c
typedef struct {
    slave_cb_t cb_buffer_tx;                    ///< 主机从共享缓冲区读取时的回调
    slave_cb_t cb_buffer_rx;                    ///< 主机向共享缓冲区写入时的回调
    slave_cb_t cb_send_dma_ready;               ///< TX数据加载到硬件(DMA)时的回调
    slave_cb_t cb_sent;                         ///< 数据发送完成时的回调
    slave_cb_t cb_recv_dma_ready;               ///< RX数据加载到硬件(DMA)时的回调
    slave_cb_t cb_recv;                         ///< 数据接收完成时的回调
    slave_cb_t cb_cmd9;                         ///< 接收到CMD9时的回调
    slave_cb_t cb_cmdA;                         ///< 接收到CMDA时的回调
    void* arg;                                  ///< 回调函数的参数
} spi_slave_hd_callback_config_t;
```

**详细成员说明：**

- **`cb_buffer_tx`** (类型: `slave_cb_t`)
  - **功能**: 主机从共享缓冲区读取时触发
  - **时机**: 主机读取共享寄存器缓冲区时
  - **应用**: 更新共享缓冲区内容

- **`cb_buffer_rx`** (类型: `slave_cb_t`)
  - **功能**: 主机向共享缓冲区写入时触发
  - **时机**: 主机写入共享寄存器缓冲区时
  - **应用**: 读取主机写入的数据

- **`cb_send_dma_ready`** (类型: `slave_cb_t`)
  - **功能**: TX数据加载到DMA时触发
  - **时机**: 数据被加载到DMA硬件准备发送时
  - **应用**: 准备下一批数据

- **`cb_sent`** (类型: `slave_cb_t`)
  - **功能**: 数据发送完成时触发
  - **时机**: DMA发送事务完成
  - **应用**: 释放发送缓冲区或准备新数据
  - **返回值**: 返回false可阻止事务加入返回队列

- **`cb_recv_dma_ready`** (类型: `slave_cb_t`)
  - **功能**: RX数据加载到DMA时触发
  - **时机**: 接收缓冲区被加载到DMA硬件时
  - **应用**: 准备下一个接收缓冲区

- **`cb_recv`** (类型: `slave_cb_t`)
  - **功能**: 数据接收完成时触发
  - **时机**: DMA接收事务完成
  - **应用**: 处理接收到的数据
  - **返回值**: 返回false可阻止事务加入返回队列

- **`cb_cmd9`** (类型: `slave_cb_t`)
  - **功能**: 接收到CMD9命令时触发
  - **应用**: 处理特定的CMD9命令

- **`cb_cmdA`** (类型: `slave_cb_t`)
  - **功能**: 接收到CMDA命令时触发
  - **应用**: 处理特定的CMDA命令

- **`arg`** (类型: `void*`)
  - **功能**: 传递给所有回调函数的用户参数
  - **用途**: 通常指向应用程序的上下文结构

### spi_slave_hd_slot_config_t

SPI Slave HD插槽配置结构体：

```c
typedef struct {
    uint8_t mode;                               ///< SPI模式(0-3)
    uint32_t spics_io_num;                      ///< CS GPIO引脚号
    uint32_t flags;                             ///< 配置标志
    uint32_t command_bits;                      ///< 命令字段位数
    uint32_t address_bits;                      ///< 地址字段位数
    uint32_t dummy_bits;                        ///< 虚拟字段位数
    uint32_t queue_size;                        ///< 事务队列大小
    spi_dma_chan_t dma_chan;                    ///< 使用的DMA通道
    spi_slave_hd_callback_config_t cb_config;   ///< 回调配置
} spi_slave_hd_slot_config_t;
```

**详细成员说明：**

- **`mode`** (类型: `uint8_t`)
  - **功能**: SPI时钟模式
  - **取值**: 
    - 0: CPOL=0, CPHA=0
    - 1: CPOL=0, CPHA=1
    - 2: CPOL=1, CPHA=0
    - 3: CPOL=1, CPHA=1
  - **说明**: 必须与主机模式匹配

- **`spics_io_num`** (类型: `uint32_t`)
  - **功能**: CS片选引脚的GPIO编号
  - **方向**: 输入引脚
  - **说明**: 由主机控制

- **`flags`** (类型: `uint32_t`)
  - **功能**: 配置标志位
  - **可用标志**:
    - `SPI_SLAVE_HD_TXBIT_LSBFIRST`: 发送LSB优先
    - `SPI_SLAVE_HD_RXBIT_LSBFIRST`: 接收LSB优先
    - `SPI_SLAVE_HD_BIT_LSBFIRST`: 收发都LSB优先
    - `SPI_SLAVE_HD_APPEND_MODE`: 启用DMA追加模式

- **`command_bits`** (类型: `uint32_t`)
  - **功能**: 命令字段的位数
  - **要求**: 8的倍数且至少为8
  - **说明**: 0表示无命令阶段

- **`address_bits`** (类型: `uint32_t`)
  - **功能**: 地址字段的位数
  - **要求**: 8的倍数且至少为8
  - **说明**: 0表示无地址阶段

- **`dummy_bits`** (类型: `uint32_t`)
  - **功能**: 虚拟字段的位数
  - **要求**: 8的倍数且至少为8
  - **说明**: 用于补偿时序

- **`queue_size`** (类型: `uint32_t`)
  - **功能**: 事务队列大小
  - **说明**: 决定可以同时排队的事务数量
  - **建议**: 根据应用需求设置（如4-8）

- **`dma_chan`** (类型: `spi_dma_chan_t`)
  - **功能**: 使用的DMA通道
  - **ESP32S2**: 可设置为主机ID或`SPI_DMA_CH_AUTO`
  - **GDMA芯片**: 必须设置为`SPI_DMA_CH_AUTO`
  - **注意**: SPI Slave HD必须使用DMA

- **`cb_config`** (类型: `spi_slave_hd_callback_config_t`)
  - **功能**: 回调函数配置
  - **说明**: 包含所有事件的回调函数指针

---

## 1. SPI Slave HD初始化和反初始化函数

### spi_slave_hd_init
**函数原型：**
```c
esp_err_t spi_slave_hd_init(spi_host_device_t host_id, const spi_bus_config_t *bus_config, const spi_slave_hd_slot_config_t *config);
```
**参数：**
host_id：要使用的SPI主机(SPI2_HOST或SPI3_HOST)
bus_config：总线配置（引脚、DMA等）
config：SPI Slave HD插槽配置
**返回值：**
ESP_OK：成功初始化
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_INVALID_STATE：函数在无效状态下调用，可能某些资源已在使用中
ESP_ERR_NOT_FOUND：没有可用的DMA通道
ESP_ERR_NO_MEM：内存分配失败
或esp_intr_alloc的其他返回值
**作用：**初始化SPI Slave HD驱动并默认启用它。分配必要的资源，配置GPIO引脚，设置DMA通道，并注册中断处理程序

**重要说明：**
- DMA对SPI Slave HD驱动是强制要求的
- ESP32S2: dma_chan可以是host_id或SPI_DMA_CH_AUTO
- 支持GDMA的芯片: dma_chan必须是SPI_DMA_CH_AUTO
- 如果CONFIG_SPI_SLAVE_ISR_IN_IRAM未设置，不能使用ESP_INTR_FLAG_IRAM标志

### spi_slave_hd_deinit
**函数原型：**
```c
esp_err_t spi_slave_hd_deinit(spi_host_device_t host_id);
```
**参数：**
host_id：要反初始化的主机
**返回值：**
ESP_OK：成功反初始化
ESP_ERR_INVALID_ARG：host_id不正确
**作用：**反初始化SPI Slave HD驱动，释放所有分配的资源，包括队列、信号量、中断处理程序、DMA通道等

**注意事项：**
- 确保没有正在进行的事务
- 释放后主机句柄不再有效
- 自动释放所有内部分配的内存

---

## 2. SPI Slave HD使能控制函数

### spi_slave_hd_enable
**函数原型：**
```c
esp_err_t spi_slave_hd_enable(spi_host_device_t host_id);
```
**参数：**
host_id：要启用的SPI外设
**返回值：**
ESP_OK：成功启用
ESP_ERR_INVALID_ARG：不支持的host_id
ESP_ERR_INVALID_STATE：外设已启用
**作用：**为已初始化的SPI主机启用SPI Slave HD功能

**注意事项：**
- spi_slave_hd_init()后不需要额外调用此函数，因为初始化时已启用
- 用于在禁用后重新启用
- 启用后可以进行数据传输

### spi_slave_hd_disable
**函数原型：**
```c
esp_err_t spi_slave_hd_disable(spi_host_device_t host_id);
```
**参数：**
host_id：要禁用的SPI外设
**返回值：**
ESP_OK：成功禁用
ESP_ERR_INVALID_ARG：不支持的host_id
ESP_ERR_INVALID_STATE：外设已禁用
**作用：**禁用已初始化的SPI主机的SPI Slave HD功能

**注意事项：**
- 禁用后不能进行数据传输
- 可以通过spi_slave_hd_enable()重新启用
- 用于临时停止SPI操作以节省功耗

---

## 3. SPI Slave HD段模式传输函数

### spi_slave_hd_queue_trans
**函数原型：**
```c
esp_err_t spi_slave_hd_queue_trans(spi_host_device_t host_id, spi_slave_chan_t chan, spi_slave_hd_data_t* trans, TickType_t timeout);
```
**参数：**
host_id：要排队事务的主机
chan：SPI_SLAVE_CHAN_TX或SPI_SLAVE_CHAN_RX
trans：事务描述符指针
timeout：数据排队前的超时时间
**返回值：**
ESP_OK：成功排队
ESP_ERR_INVALID_ARG：输入参数无效，可能的原因：
  - 提供的缓冲区不是DMA可访问的
  - 数据长度无效（不大于0，或超过最大传输长度）
  - 事务方向无效
ESP_ERR_TIMEOUT：超时前无法排队数据，主机仍在处理前一个事务
ESP_ERR_INVALID_STATE：在无效状态下调用函数，此API应在段模式下调用
**作用：**在段模式下排队一个事务。主机将在CS有效且命令/地址阶段匹配时处理此事务

**使用说明：**
- 仅在段模式下使用（未设置SPI_SLAVE_HD_APPEND_MODE标志）
- 缓冲区必须是DMA可访问的
- 数据长度必须在有效范围内
- 通过spi_slave_hd_get_trans_res()获取结果

### spi_slave_hd_get_trans_res
**函数原型：**
```c
esp_err_t spi_slave_hd_get_trans_res(spi_host_device_t host_id, spi_slave_chan_t chan, spi_slave_hd_data_t **out_trans, TickType_t timeout);
```
**参数：**
host_id：要获取事务结果的主机
chan：通道，SPI_SLAVE_CHAN_TX或SPI_SLAVE_CHAN_RX
out_trans：指向事务描述符指针的指针（输出）
timeout：获取结果前的超时时间
**返回值：**
ESP_OK：成功获取结果
ESP_ERR_INVALID_ARG：函数无效
ESP_ERR_TIMEOUT：超时前没有完成的事务
ESP_ERR_INVALID_STATE：在无效状态下调用函数，此API应在段模式下调用
**作用：**获取数据事务的结果（段模式）。硬件已完成此事务，对于RX方向，成员trans_len表示实际接收的字节数；对于TX方向，trans_len无意义

**注意事项：**
- 此API调用成功次数应与spi_slave_hd_queue_trans相同
- 这是阻塞函数，会等待直到事务完成
- 接收事务中，trans_len指示实际接收的数据量

---

## 4. SPI Slave HD共享缓冲区访问函数

### spi_slave_hd_read_buffer
**函数原型：**
```c
void spi_slave_hd_read_buffer(spi_host_device_t host_id, int addr, uint8_t *out_data, size_t len);
```
**参数：**
host_id：要读取共享寄存器的主机
addr：要读取的寄存器地址，0到SOC_SPI_MAXIMUM_BUFFER_SIZE-1
out_data：存储读取数据的输出缓冲区
len：要读取的长度，不大于SOC_SPI_MAXIMUM_BUFFER_SIZE-addr
**返回值：**
无
**作用：**读取共享寄存器缓冲区。这些寄存器可以被主机和从机同时访问，用于交换少量控制数据或状态信息

**使用说明：**
- 共享缓冲区大小由SOC_SPI_MAXIMUM_BUFFER_SIZE定义
- 地址范围：0 ~ SOC_SPI_MAXIMUM_BUFFER_SIZE-1
- 可以在任何时候读取，无需等待事务完成
- 通常用于读取主机写入的命令或配置

### spi_slave_hd_write_buffer
**函数原型：**
```c
void spi_slave_hd_write_buffer(spi_host_device_t host_id, int addr, uint8_t *data, size_t len);
```
**参数：**
host_id：要写入共享寄存器的主机
addr：要写入的寄存器地址，0到SOC_SPI_MAXIMUM_BUFFER_SIZE-1
data：要写入的数据缓冲区
len：要写入的长度，不大于SOC_SPI_MAXIMUM_BUFFER_SIZE-addr
**返回值：**
无
**作用：**写入共享寄存器缓冲区。主机可以通过SPI读取命令访问这些数据

**使用说明：**
- 共享缓冲区大小由SOC_SPI_MAXIMUM_BUFFER_SIZE定义
- 地址范围：0 ~ SOC_SPI_MAXIMUM_BUFFER_SIZE-1
- 可以在任何时候写入，无需等待事务完成
- 通常用于向主机提供状态信息或响应数据

---

## 5. SPI Slave HD追加模式传输函数

### spi_slave_hd_append_trans
**函数原型：**
```c
esp_err_t spi_slave_hd_append_trans(spi_host_device_t host_id, spi_slave_chan_t chan, spi_slave_hd_data_t *trans, TickType_t timeout);
```
**参数：**
host_id：要加载事务的主机
chan：SPI_SLAVE_CHAN_TX或SPI_SLAVE_CHAN_RX
trans：事务描述符指针
timeout：事务加载前的超时时间
**返回值：**
ESP_OK：成功加载事务
ESP_ERR_INVALID_ARG：输入参数无效，可能的原因：
  - 提供的缓冲区不是DMA可访问的
  - 数据长度无效（不大于0，或超过最大传输长度）
  - 事务方向无效
ESP_ERR_TIMEOUT：主机仍在处理前一个事务，没有可用的事务供从机加载
ESP_ERR_INVALID_STATE：在无效状态下调用函数，此API应在追加模式下调用
**作用：**在追加模式下加载事务。在此模式下，用户事务描述符将被追加到DMA，DMA将持续处理数据而不停止

**重要说明：**
- 仅在追加模式下使用（设置了SPI_SLAVE_HD_APPEND_MODE标志）
- 当前仅支持数据长度在4092字节以内的事务
- 缓冲区必须是DMA可访问的
- DMA会自动链接新的事务，无需停止当前传输

### spi_slave_hd_get_append_trans_res
**函数原型：**
```c
esp_err_t spi_slave_hd_get_append_trans_res(spi_host_device_t host_id, spi_slave_chan_t chan, spi_slave_hd_data_t **out_trans, TickType_t timeout);
```
**参数：**
host_id：要加载事务的主机
chan：SPI_SLAVE_CHAN_TX或SPI_SLAVE_CHAN_RX
out_trans：指向事务描述符指针的指针（输出）
timeout：获取结果前的超时时间
**返回值：**
ESP_OK：成功获取结果
ESP_ERR_INVALID_ARG：函数无效
ESP_ERR_TIMEOUT：超时前没有完成的事务
ESP_ERR_INVALID_STATE：在无效状态下调用函数，此API应在追加模式下调用
**作用：**获取数据事务的结果（追加模式）。硬件已完成此事务，对于RX方向，成员trans_len表示实际接收的字节数；对于TX方向，trans_len无意义

**注意事项：**
- 此API调用成功次数应与spi_slave_hd_append_trans相同
- 这是阻塞函数，会等待直到事务完成
- 追加模式可以实现连续的数据流传输

---

## 6. SPI Slave HD使用注意事项

### 6.1 工作模式选择

1. **段模式 vs 追加模式**
   - 段模式：每次传输是独立的段，传输间有明确的开始和结束
   - 追加模式：DMA持续运行，新事务动态追加到队列中
   - 段模式适用于离散的、独立的数据包传输
   - 追加模式适用于连续的数据流传输

2. **DMA要求**
   - SPI Slave HD必须使用DMA
   - ESP32S2: 可以指定DMA通道
   - GDMA芯片: 必须使用SPI_DMA_CH_AUTO

3. **模式配置**
   - 通过flags中的SPI_SLAVE_HD_APPEND_MODE标志选择模式
   - 两种模式使用不同的API函数
   - 初始化后不能更改模式

### 6.2 内存管理

4. **DMA缓冲区要求**
   - 所有数据缓冲区必须是DMA可访问的
   - 使用heap_caps_malloc(size, MALLOC_CAP_DMA)分配
   - 不能使用栈上的局部变量
   - 不能使用flash中的常量数据

5. **自动对齐**
   - 设置SPI_SLAVE_HD_TRANS_DMA_BUFFER_ALIGN_AUTO标志
   - 驱动会自动重新分配对齐的缓冲区
   - 会消耗额外的内存和性能
   - 建议手动管理对齐以获得最佳性能

6. **缓冲区大小**
   - 接收缓冲区应为4字节的倍数
   - 超出部分会被截断
   - 追加模式当前限制单次传输不超过4092字节
   - 不能超过max_transfer_sz配置

### 6.3 事务管理

7. **事务排队**
   - queue_size决定可排队的事务数量
   - 队列满时函数会阻塞直到超时
   - 合理设置超时时间避免死锁

8. **事务结果获取**
   - get_trans_res调用次数必须与queue_trans或append_trans相同
   - 接收事务的trans_len表示实际接收的字节数
   - 发送事务的trans_len无意义

9. **事务生命周期**
   - 排队 → DMA加载 → 传输 → 完成 → 返回队列
   - 每个阶段都可以设置回调函数
   - 在获取结果前不要修改事务描述符

### 6.4 回调函数

10. **回调上下文**
    - 所有回调在ISR上下文中运行
    - 必须快速执行，不能阻塞
    - 不能调用FreeRTOS的阻塞API
    - 使用FromISR版本的API

11. **回调返回值**
    - cb_sent和cb_recv可以返回false
    - 返回false阻止事务加入返回队列
    - 用户需要自己管理事务生命周期
    - 返回true让驱动自动管理

12. **IRAM放置**
    - 设置CONFIG_SPI_SLAVE_ISR_IN_IRAM
    - 将ISR和回调放入IRAM
    - 提高性能，减少cache miss
    - 增加IRAM使用量

### 6.5 共享缓冲区

13. **共享寄存器用途**
    - 交换少量控制数据或状态信息
    - 大小由SOC_SPI_MAXIMUM_BUFFER_SIZE定义
    - 主机和从机都可以随时访问
    - 不需要DMA传输

14. **读写时机**
    - 可以在任何时候读写
    - 不受事务状态影响
    - 建议使用回调同步访问
    - 避免同时读写同一地址

15. **缓冲区地址**
    - 地址范围：0 ~ SOC_SPI_MAXIMUM_BUFFER_SIZE-1
    - 读写长度不能超出范围
    - 可以分段使用（如命令区、数据区）

### 6.6 时序和同步

16. **SPI模式匹配**
    - mode必须与主机配置匹配
    - CPOL和CPHA决定时钟极性和采样边沿
    - 不匹配会导致数据错误

17. **命令和地址**
    - command_bits、address_bits、dummy_bits必须是8的倍数
    - 至少为8位（如果使用）
    - 0表示不使用该阶段
    - 必须与主机配置一致

18. **CS信号**
    - CS由主机控制
    - 从机监听CS的下降沿开始传输
    - CS上升沿结束传输
    - 确保CS引脚配置正确

### 6.7 电源管理

19. **PM锁**
    - 驱动自动管理PM锁
    - 初始化时获取锁
    - 禁用时释放锁
    - 确保SPI时钟稳定

20. **睡眠保持**
    - flags中的allow_pd标志
    - 允许在轻度睡眠时掉电
    - 驱动会备份/恢复寄存器
    - 需要CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP支持

### 6.8 性能优化

21. **DMA描述符数量**
    - 由max_transfer_sz决定
    - 更多描述符支持更大的传输
    - 消耗更多内存
    - 根据应用需求平衡

22. **队列深度**
    - 增加queue_size支持更多并发事务
    - 减少主机等待时间
    - 消耗更多内存
    - 典型值：4-8

23. **追加模式优势**
    - 无需停止DMA即可加载新事务
    - 适用于连续数据流
    - 减少传输间隙
    - 提高吞吐量

### 6.9 错误处理

24. **超时处理**
    - 所有阻塞函数都有超时参数
    - 合理设置超时避免死锁
    - portMAX_DELAY表示永不超时
    - 检查返回值ESP_ERR_TIMEOUT

25. **参数验证**
    - 驱动会检查缓冲区是否DMA可访问
    - 检查数据长度是否有效
    - 检查通道参数是否正确
    - 返回ESP_ERR_INVALID_ARG表示参数错误

26. **状态检查**
    - 确保在正确的模式下调用API
    - 段模式和追加模式API不能混用
    - 返回ESP_ERR_INVALID_STATE表示状态错误

### 6.10 调试和测试

27. **日志输出**
    - 使用ESP_LOG宏输出调试信息
    - 设置适当的日志级别
    - 在回调中避免大量日志输出

28. **事务跟踪**
    - 使用trans->arg字段
    - 在回调中识别特定事务
    - 便于调试和性能分析

29. **硬件限制**
    - 了解芯片特定的限制
    - SOC_SPI_MAXIMUM_BUFFER_SIZE
    - SPI_MAX_DMA_LEN
    - 查阅芯片技术手册

30. **与主机配合**
    - 确保主机和从机配置一致
    - 协商好命令、地址格式
    - 定义清晰的通信协议
    - 处理好异常情况