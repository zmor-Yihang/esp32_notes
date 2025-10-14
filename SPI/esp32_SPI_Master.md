# ESP32 SPI Master API 完整函数参考

## 0. SPI Master主要数据结构

### spi_device_interface_config_t

SPI从设备接口配置结构体，用于配置连接到SPI总线的从设备：

```c
typedef struct {
    uint8_t command_bits;           ///< 命令阶段的位数，常用值为0或8
    uint8_t address_bits;           ///< 地址阶段的位数，常用值为0、8、16、24或32
    uint8_t dummy_bits;             ///< 虚拟周期的位数，用于补偿时序
    uint8_t mode;                   ///< SPI模式(0-3)
    spi_clock_source_t clock_source;///< 时钟源选择
    uint16_t duty_cycle_pos;        ///< 占空比(1-256)，256表示50%占空比
    uint16_t cs_ena_pretrans;       ///< CS在传输前的准备时间(SPI时钟周期数)
    uint8_t cs_ena_posttrans;       ///< CS在传输后保持有效的时间(SPI时钟周期数)
    int clock_speed_hz;             ///< SPI时钟频率(Hz)
    int input_delay_ns;             ///< 输入延迟补偿(纳秒)，用于高频通信
    spi_sampling_point_t sample_point;  ///< 采样点选择
    int spics_io_num;               ///< CS GPIO引脚编号，-1表示不使用
    uint32_t flags;                 ///< 位掩码标志，SPI_DEVICE_*常量
    int queue_size;                 ///< 事务队列大小，决定可以排队的事务数
    transaction_cb_t pre_cb;        ///< 事务前回调函数
    transaction_cb_t post_cb;       ///< 事务后回调函数
} spi_device_interface_config_t;
```

**详细成员说明：**

- **`command_bits`** (类型: `uint8_t`)
  - **功能**: 命令阶段的位数
  - **常用值**: 0(无命令)、8(8位命令)
  - **说明**: 命令总是在单线模式下传输

- **`address_bits`** (类型: `uint8_t`)
  - **功能**: 地址阶段的位数
  - **常用值**: 0、8、16、24、32
  - **说明**: 地址可以在单线或多线模式下传输

- **`dummy_bits`** (类型: `uint8_t`)
  - **功能**: 虚拟周期的位数
  - **作用**: 用于补偿输入延迟，在高频时钟下特别重要
  - **范围**: 0-255

- **`mode`** (类型: `uint8_t`)
  - **功能**: SPI时钟模式
  - **取值**: 
    - 0: CPOL=0, CPHA=0
    - 1: CPOL=0, CPHA=1
    - 2: CPOL=1, CPHA=0
    - 3: CPOL=1, CPHA=1

- **`clock_source`** (类型: `spi_clock_source_t`)
  - **功能**: 时钟源选择
  - **说明**: 不同时钟源影响可用频率范围和功耗

- **`duty_cycle_pos`** (类型: `uint16_t`)
  - **功能**: 时钟占空比
  - **范围**: 1-256
  - **默认**: 128(50%占空比)

- **`cs_ena_pretrans`** (类型: `uint16_t`)
  - **功能**: CS在传输前的准备时间
  - **单位**: SPI时钟周期数
  - **用途**: 某些设备需要CS提前拉低

- **`cs_ena_posttrans`** (类型: `uint8_t`)
  - **功能**: CS在传输后保持有效的时间
  - **单位**: SPI时钟周期数
  - **用途**: 某些设备需要CS延迟拉高

- **`clock_speed_hz`** (类型: `int`)
  - **功能**: SPI时钟频率
  - **单位**: Hz
  - **说明**: 实际频率可能因分频器限制而略有不同

- **`input_delay_ns`** (类型: `int`)
  - **功能**: 输入延迟补偿
  - **单位**: 纳秒
  - **用途**: ESP32高频通信时补偿GPIO矩阵延迟

- **`sample_point`** (类型: `spi_sampling_point_t`)
  - **功能**: 采样点选择
  - **说明**: 控制何时采样MISO数据

- **`spics_io_num`** (类型: `int`)
  - **功能**: CS引脚GPIO编号
  - **取值**: 有效GPIO或-1(不使用硬件CS)

- **`flags`** (类型: `uint32_t`)
  - **功能**: 设备特性标志
  - **可用标志**:
    - `SPI_DEVICE_TXBIT_LSBFIRST`: 发送LSB优先
    - `SPI_DEVICE_RXBIT_LSBFIRST`: 接收LSB优先
    - `SPI_DEVICE_BIT_LSBFIRST`: 收发都LSB优先
    - `SPI_DEVICE_3WIRE`: 三线模式(MOSI用于收发)
    - `SPI_DEVICE_POSITIVE_CS`: CS高电平有效
    - `SPI_DEVICE_HALFDUPLEX`: 半双工模式
    - `SPI_DEVICE_CLK_AS_CS`: 时钟信号用作CS
    - `SPI_DEVICE_NO_DUMMY`: 禁用自动插入虚拟位
    - `SPI_DEVICE_DDRCLK`: DDR模式时钟
    - `SPI_DEVICE_NO_RETURN_RESULT`: 不返回事务描述符

- **`queue_size`** (类型: `int`)
  - **功能**: 事务队列大小
  - **说明**: 决定可以排队等待的事务数量

- **`pre_cb`** (类型: `transaction_cb_t`)
  - **功能**: 事务前回调函数
  - **时机**: 在事务开始前调用
  - **上下文**: 在ISR中运行

- **`post_cb`** (类型: `transaction_cb_t`)
  - **功能**: 事务后回调函数
  - **时机**: 在事务完成后调用
  - **上下文**: 在ISR中运行

### spi_transaction_t

SPI事务描述符结构体，描述一次SPI传输：

```c
struct spi_transaction_t {
    uint32_t flags;                 ///< 位掩码标志，SPI_TRANS_*常量
    uint16_t cmd;                   ///< 命令数据
    uint64_t addr;                  ///< 地址数据
    size_t length;                  ///< 发送数据的总位数
    size_t rxlength;                ///< 接收数据的总位数，0表示与length相同
    uint32_t override_freq_hz;      ///< 临时覆盖时钟频率(Hz)，0表示使用设备配置
    void *user;                     ///< 用户定义的数据指针
    union {
        const void *tx_buffer;      ///< 指向发送缓冲区的指针
        uint8_t tx_data[4];         ///< 直接存储最多4字节的发送数据
    };
    union {
        void *rx_buffer;            ///< 指向接收缓冲区的指针
        uint8_t rx_data[4];         ///< 直接存储最多4字节的接收数据
    };
};
```

**详细成员说明：**

- **`flags`** (类型: `uint32_t`)
  - **功能**: 事务特性标志
  - **可用标志**:
    - `SPI_TRANS_MODE_DIO`: 双线模式
    - `SPI_TRANS_MODE_QIO`: 四线模式
    - `SPI_TRANS_MODE_OCT`: 八线模式
    - `SPI_TRANS_USE_RXDATA`: 使用rx_data而非rx_buffer
    - `SPI_TRANS_USE_TXDATA`: 使用tx_data而非tx_buffer
    - `SPI_TRANS_MODE_DIOQIO_ADDR`: 地址也使用多线模式
    - `SPI_TRANS_VARIABLE_CMD`: 使用可变命令位数
    - `SPI_TRANS_VARIABLE_ADDR`: 使用可变地址位数
    - `SPI_TRANS_VARIABLE_DUMMY`: 使用可变虚拟位数
    - `SPI_TRANS_CS_KEEP_ACTIVE`: 传输后保持CS有效
    - `SPI_TRANS_MULTILINE_CMD`: 命令也使用多线模式
    - `SPI_TRANS_MULTILINE_ADDR`: 地址也使用多线模式
    - `SPI_TRANS_DMA_BUFFER_ALIGN_MANUAL`: 手动管理DMA对齐

- **`cmd`** (类型: `uint16_t`)
  - **功能**: 命令数据
  - **位数**: 由设备配置的command_bits决定
  - **传输**: 总是MSB优先，单线模式

- **`addr`** (类型: `uint64_t`)
  - **功能**: 地址数据
  - **位数**: 由设备配置的address_bits决定
  - **传输**: 总是MSB优先

- **`length`** (类型: `size_t`)
  - **功能**: 发送数据的总位数
  - **说明**: 可以不是字节对齐

- **`rxlength`** (类型: `size_t`)
  - **功能**: 接收数据的总位数
  - **默认**: 0表示与length相同
  - **说明**: 半双工模式下可以与length不同

- **`override_freq_hz`** (类型: `uint32_t`)
  - **功能**: 临时覆盖时钟频率
  - **单位**: Hz
  - **说明**: 0表示使用设备配置的频率

- **`user`** (类型: `void*`)
  - **功能**: 用户定义的数据指针
  - **用途**: 在回调中识别事务

- **`tx_buffer / tx_data`** (联合体)
  - **tx_buffer**: 指向DMA可访问内存的发送缓冲区
  - **tx_data**: 直接存储≤4字节的数据
  - **选择**: 通过SPI_TRANS_USE_TXDATA标志控制

- **`rx_buffer / rx_data`** (联合体)
  - **rx_buffer**: 指向DMA可访问内存的接收缓冲区
  - **rx_data**: 直接存储≤4字节的数据
  - **选择**: 通过SPI_TRANS_USE_RXDATA标志控制

### spi_transaction_ext_t

扩展的SPI事务描述符，支持可变长度的命令和地址：

```c
typedef struct {
    struct spi_transaction_t base;  ///< 基本事务结构
    uint8_t command_bits;           ///< 此事务的命令位数
    uint8_t address_bits;           ///< 此事务的地址位数
    uint8_t dummy_bits;             ///< 此事务的虚拟位数
} spi_transaction_ext_t;
```

**使用说明：**
- 必须在base.flags中设置对应的`SPI_TRANS_VARIABLE_*`标志
- 允许同一设备的不同事务使用不同的命令/地址长度

### transaction_cb_t

事务回调函数类型定义：

```c
typedef void(*transaction_cb_t)(spi_transaction_t *trans);
```

**参数：**
- trans: 事务描述符指针

**说明：**
- 在ISR上下文中调用
- 必须快速执行，不能阻塞
- 不能调用FreeRTOS阻塞API

### spi_device_handle_t

SPI设备句柄类型：

```c
typedef struct spi_device_t *spi_device_handle_t;
```

**说明：**
- 不透明的设备句柄
- 由`spi_bus_add_device()`返回
- 用于所有设备相关的API调用

---

## 1. SPI设备管理函数

### spi_bus_add_device
**函数原型：**
```c
esp_err_t spi_bus_add_device(spi_host_device_t host_id, const spi_device_interface_config_t *dev_config, spi_device_handle_t *handle);
```
**参数：**
host_id：要在其上分配设备的SPI外设(SPI2_HOST或SPI3_HOST)
dev_config：指向spi_device_interface_config_t结构体的指针，指定设备的接口协议配置
handle：指向变量的指针，用于保存设备句柄
**返回值：**
ESP_OK：成功添加设备
ESP_ERR_INVALID_ARG：参数无效或不支持的配置组合(例如，启用SPI_DEVICE_NO_RETURN_RESULT但未设置post_cb)
ESP_ERR_INVALID_STATE：选定的时钟源不可用或SPI总线未初始化
ESP_ERR_NOT_FOUND：主机没有空闲的CS插槽
ESP_ERR_NO_MEM：内存不足
**作用：**在SPI总线上分配一个设备。初始化设备的内部结构，在指定的SPI主机外设上分配CS引脚并将其路由到指定的GPIO。所有SPI主机设备有三个CS引脚，因此最多可以控制三个设备

**重要说明：**
- 在ESP32上，由于GPIO矩阵的延迟，SPI主机能正确采样从设备输出的最大频率低于使用IOMUX的情况
- 典型的最大频率：80MHz(IOMUX引脚)和26MHz(GPIO矩阵引脚)
- 在半双工模式下，通过正确设置dev_config中的input_delay_ns，可以用额外的虚拟周期补偿延迟
- 其他芯片(非ESP32)没有明显延迟

### spi_bus_remove_device
**函数原型：**
```c
esp_err_t spi_bus_remove_device(spi_device_handle_t handle);
```
**参数：**
handle：要释放的设备句柄
**返回值：**
ESP_OK：成功移除设备
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_INVALID_STATE：设备已经被释放
**作用：**从SPI总线移除一个设备，释放CS引脚和其他资源

**注意事项：**
- 确保设备上没有正在进行的事务
- 移除后设备句柄不再有效

### spi_device_get_actual_freq
**函数原型：**
```c
esp_err_t spi_device_get_actual_freq(spi_device_handle_t handle, int* freq_khz);
```
**参数：**
handle：设备句柄
freq_khz：返回实际频率(kHz)的指针
**返回值：**
ESP_OK：成功获取频率
ESP_ERR_INVALID_ARG：参数无效
**作用：**获取设备的实际时钟频率。由于分频器限制，实际频率可能与配置的频率略有不同

---

## 2. SPI时序计算和频率限制函数

### spi_get_actual_clock (已弃用)
**函数原型：**
```c
int spi_get_actual_clock(int fapb, int hz, int duty_cycle) __attribute__((deprecated));
```
**参数：**
fapb：APB时钟频率，应为APB_CLK_FREQ
hz：期望的工作频率
duty_cycle：SPI时钟占空比
**返回值：**
返回最接近期望频率的实际工作频率
**作用：**计算最接近期望频率的实际工作频率

**注意：**此函数已弃用，请使用spi_device_get_actual_freq()代替

### spi_get_timing
**函数原型：**
```c
void spi_get_timing(bool gpio_is_used, int input_delay_ns, int eff_clk, int *dummy_o, int *cycles_remain_o);
```
**参数：**
gpio_is_used：如果使用GPIO矩阵为True，如果使用IOMUX引脚为False
input_delay_ns：从SCLK启动边沿到MISO数据有效的输入延迟(纳秒)
eff_clk：从spi_get_actual_clock()获得的有效时钟频率(Hz)
dummy_o：用于输出使用的虚拟位的地址，如果不需要设为NULL
cycles_remain_o：剩余周期(使用虚拟位后)的输出地址
  - -1：如果剩余周期太多，建议补偿半个时钟
  - 0：如果没有剩余周期或未使用虚拟位
  - 正值：建议补偿的周期数
**返回值：**
无
**作用：**计算指定频率和设置的时序参数

**注意：**如果*dummy_o不为零，表示应在半双工模式下应用虚拟位，全双工模式可能无法工作

### spi_get_freq_limit
**函数原型：**
```c
int spi_get_freq_limit(bool gpio_is_used, int input_delay_ns);
```
**参数：**
gpio_is_used：如果使用GPIO矩阵为True，如果使用原生引脚为False
input_delay_ns：从SCLK启动边沿到MISO数据有效的输入延迟(纳秒)
**返回值：**
当前配置的频率限制
**作用：**获取当前配置的频率限制。SPI主机在此限制下工作正常，超过限制时，全双工模式和DMA将无法工作，半双工模式会应用虚拟位

**使用说明：**
- 此函数帮助确定在特定硬件配置下可以安全使用的最大频率
- 超过此频率可能需要切换到半双工模式或使用虚拟周期

### spi_bus_get_max_transaction_len
**函数原型：**
```c
esp_err_t spi_bus_get_max_transaction_len(spi_host_device_t host_id, size_t *max_bytes);
```
**参数：**
host_id：SPI外设(SPI2_HOST或SPI3_HOST)
max_bytes：返回单次事务最大长度(字节)的指针
**返回值：**
ESP_OK：成功获取
ESP_ERR_INVALID_ARG：参数无效
**作用：**获取单次事务的最大长度(字节)

**注意事项：**
- 返回值取决于总线初始化时配置的DMA通道
- 使用DMA时通常可以传输更大的数据

---

## 3. SPI中断传输函数

### spi_device_queue_trans
**函数原型：**
```c
esp_err_t spi_device_queue_trans(spi_device_handle_t handle, spi_transaction_t *trans_desc, TickType_t ticks_to_wait);
```
**参数：**
handle：使用spi_bus_add_device()获得的设备句柄
trans_desc：要执行的事务描述符
ticks_to_wait：等待队列有空间的超时时间，使用portMAX_DELAY表示永不超时
**返回值：**
ESP_OK：成功排队事务
ESP_ERR_INVALID_ARG：参数无效，或指定SPI_TRANS_CS_KEEP_ACTIVE但未获取总线，或设置SPI_TRANS_DMA_BUFFER_ALIGN_MANUAL但缓冲区不符合DMA要求
ESP_ERR_TIMEOUT：在ticks_to_wait过期前队列没有空间
ESP_ERR_NO_MEM：分配DMA临时缓冲区失败
ESP_ERR_INVALID_STATE：先前的事务未完成
**作用：**将SPI事务排队进行中断执行。通过spi_device_get_trans_result()获取结果

**重要说明：**
- 通常设备不能同时排队轮询和中断事务
- 这是异步函数，立即返回
- 事务完成后通过spi_device_get_trans_result()获取结果

### spi_device_get_trans_result
**函数原型：**
```c
esp_err_t spi_device_get_trans_result(spi_device_handle_t handle, spi_transaction_t **trans_desc, TickType_t ticks_to_wait);
```
**参数：**
handle：使用spi_bus_add_device()获得的设备句柄
trans_desc：指向能够包含已执行事务描述符指针的变量的指针
ticks_to_wait：等待返回项的超时时间，使用portMAX_DELAY表示永不超时
**返回值：**
ESP_OK：成功获取结果
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_NOT_SUPPORTED：设置了SPI_DEVICE_NO_RETURN_RESULT标志
ESP_ERR_TIMEOUT：在ticks_to_wait过期前没有完成的事务
**作用：**获取之前由spi_device_queue_trans排队的SPI事务的结果。此例程会等待直到设备的事务成功完成，然后返回事务描述符

**注意事项：**
- 此函数会阻塞直到事务完成
- 返回的描述符不应在事务完成前修改
- 可以用于检查结果、释放内存或重用缓冲区

### spi_device_transmit
**函数原型：**
```c
esp_err_t spi_device_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);
```
**参数：**
handle：使用spi_bus_add_device()获得的设备句柄
trans_desc：要执行的事务描述符
**返回值：**
ESP_OK：成功完成传输
ESP_ERR_INVALID_ARG：参数无效
**作用：**发送SPI事务，等待其完成并返回结果。此函数等效于先调用spi_device_queue_trans()再调用spi_device_get_trans_result()。当还有从spi_device_queue_trans()或polling_start/transmit单独排队(启动)的未完成事务时，不要使用此函数

**注意事项：**
- 这是阻塞函数
- 多个任务访问同一SPI设备时不是线程安全的
- 通常设备不能同时启动轮询和中断事务

---

## 4. SPI轮询传输函数

### spi_device_polling_start
**函数原型：**
```c
esp_err_t spi_device_polling_start(spi_device_handle_t handle, spi_transaction_t *trans_desc, TickType_t ticks_to_wait);
```
**参数：**
handle：使用spi_bus_add_device()获得的设备句柄
trans_desc：要执行的事务描述符
ticks_to_wait：等待队列有空间的超时时间，目前仅支持portMAX_DELAY
**返回值：**
ESP_OK：成功启动传输
ESP_ERR_INVALID_ARG：参数无效，或指定SPI_TRANS_CS_KEEP_ACTIVE但未获取总线，或设置SPI_TRANS_DMA_BUFFER_ALIGN_MANUAL但缓冲区不符合DMA要求
ESP_ERR_TIMEOUT：设备在ticks_to_wait过期前无法获得总线控制权
ESP_ERR_NO_MEM：分配DMA临时缓冲区失败
ESP_ERR_INVALID_STATE：先前的事务未完成
**作用：**立即启动一个轮询事务

**重要说明：**
- 通常设备不能同时启动轮询和中断事务
- 设备不能在另一个轮询事务未完成时启动新的轮询事务
- 这是非阻塞函数，启动事务后立即返回

### spi_device_polling_end
**函数原型：**
```c
esp_err_t spi_device_polling_end(spi_device_handle_t handle, TickType_t ticks_to_wait);
```
**参数：**
handle：使用spi_bus_add_device()获得的设备句柄
ticks_to_wait：等待返回项的超时时间，使用portMAX_DELAY表示永不超时
**返回值：**
ESP_OK：成功完成传输
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_TIMEOUT：事务在ticks_to_wait过期前无法完成
**作用：**轮询直到轮询事务结束。此例程在事务成功完成前不会返回。任务不会被阻塞，而是主动忙等待事务完成

**注意事项：**
- 这会占用CPU直到事务完成
- 适用于非常短的事务或需要精确时序的场景

### spi_device_polling_transmit
**函数原型：**
```c
esp_err_t spi_device_polling_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);
```
**参数：**
handle：使用spi_bus_add_device()获得的设备句柄
trans_desc：要执行的事务描述符
**返回值：**
ESP_OK：成功完成传输
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_TIMEOUT：设备无法获得总线控制权
ESP_ERR_NO_MEM：分配DMA临时缓冲区失败
ESP_ERR_INVALID_STATE：同一设备的先前事务未完成
**作用：**发送轮询事务，等待其完成并返回结果。此函数等效于先调用spi_device_polling_start()再调用spi_device_polling_end()。当还有未完成的事务时不要使用此函数

**注意事项：**
- 这是阻塞函数，会占用CPU
- 多个任务访问同一SPI设备时不是线程安全的
- 通常设备不能同时启动轮询和中断事务

---

## 5. SPI总线控制函数

### spi_device_acquire_bus
**函数原型：**
```c
esp_err_t spi_device_acquire_bus(spi_device_t *device, TickType_t wait);
```
**参数：**
device：要获取总线的设备句柄
wait：等待总线的超时时间
**返回值：**
ESP_OK：成功获取总线
ESP_ERR_TIMEOUT：等待超时
**作用：**占用SPI总线供某个设备连续使用。其他设备无法使用总线，直到调用spi_device_release_bus()

**使用场景：**
- 需要连续发送多个事务而不被其他设备打断
- 使用SPI_TRANS_CS_KEEP_ACTIVE保持CS有效

### spi_device_release_bus
**函数原型：**
```c
void spi_device_release_bus(spi_device_t *dev);
```
**参数：**
dev：要释放总线的设备句柄
**返回值：**
无
**作用：**释放SPI总线，允许其他设备使用

**注意事项：**
- 必须与spi_device_acquire_bus()配对使用
- 释放后其他设备才能使用总线
