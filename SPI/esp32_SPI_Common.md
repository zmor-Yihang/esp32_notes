# ESP32 SPI Common API 完整函数参考

## 0. SPI Common主要数据结构

### spi_bus_config_t

SPI总线配置结构体，用于指定总线的GPIO引脚配置：

```c
typedef struct {
    union {
        int mosi_io_num;    ///< GPIO引脚用于主出从入(=spi_d)信号，或-1表示不使用
        int data0_io_num;   ///< 四线/八线模式下的spi data0信号GPIO引脚，或-1表示不使用
    };
    union {
        int miso_io_num;    ///< GPIO引脚用于主入从出(=spi_q)信号，或-1表示不使用
        int data1_io_num;   ///< 四线/八线模式下的spi data1信号GPIO引脚，或-1表示不使用
    };
    int sclk_io_num;        ///< GPIO引脚用于SPI时钟信号，或-1表示不使用
    union {
        int quadwp_io_num;  ///< GPIO引脚用于WP(写保护)信号，或-1表示不使用
        int data2_io_num;   ///< 四线/八线模式下的spi data2信号GPIO引脚，或-1表示不使用
    };
    union {
        int quadhd_io_num;  ///< GPIO引脚用于HD(保持)信号，或-1表示不使用
        int data3_io_num;   ///< 四线/八线模式下的spi data3信号GPIO引脚，或-1表示不使用
    };
    int data4_io_num;       ///< 八线模式下的spi data4信号GPIO引脚，或-1表示不使用
    int data5_io_num;       ///< 八线模式下的spi data5信号GPIO引脚，或-1表示不使用
    int data6_io_num;       ///< 八线模式下的spi data6信号GPIO引脚，或-1表示不使用
    int data7_io_num;       ///< 八线模式下的spi data7信号GPIO引脚，或-1表示不使用
    bool data_io_default_level; ///< 无事务时输出数据IO的默认电平
    int max_transfer_sz;    ///< 最大传输大小(字节)。启用DMA时默认为4092，禁用DMA时为SOC_SPI_MAXIMUM_BUFFER_SIZE
    uint32_t flags;         ///< 总线能力标志，SPICOMMON_BUSFLAG_*标志的或运算值
    esp_intr_cpu_affinity_t isr_cpu_id; ///< 选择注册SPI ISR的CPU核心
    int intr_flags;         ///< 总线中断标志，用于设置优先级和IRAM属性
} spi_bus_config_t;
```

**详细成员说明：**

- **`mosi_io_num / data0_io_num`** (类型: `int`)
  - **功能**: 主机输出从机输入数据线的GPIO引脚编号
  - **取值**: 有效GPIO编号或-1(不使用)
  - **说明**: 在四线/八线模式下也称为data0线

- **`miso_io_num / data1_io_num`** (类型: `int`)
  - **功能**: 主机输入从机输出数据线的GPIO引脚编号
  - **取值**: 有效GPIO编号或-1(不使用)
  - **说明**: 在四线/八线模式下也称为data1线

- **`sclk_io_num`** (类型: `int`)
  - **功能**: SPI时钟信号的GPIO引脚编号
  - **取值**: 有效GPIO编号或-1(不使用)
  - **重要性**: SPI通信的时钟源，必须配置

- **`quadwp_io_num / data2_io_num`** (类型: `int`)
  - **功能**: 写保护信号或四线模式data2线的GPIO引脚编号
  - **取值**: 有效GPIO编号或-1(不使用)
  - **用途**: 四线(Quad)模式必需

- **`quadhd_io_num / data3_io_num`** (类型: `int`)
  - **功能**: 保持信号或四线模式data3线的GPIO引脚编号
  - **取值**: 有效GPIO编号或-1(不使用)
  - **用途**: 四线(Quad)模式必需

- **`data4_io_num ~ data7_io_num`** (类型: `int`)
  - **功能**: 八线(Octal)模式下的额外数据线GPIO引脚编号
  - **取值**: 有效GPIO编号或-1(不使用)
  - **用途**: 仅八线模式使用，可提供更高带宽

- **`data_io_default_level`** (类型: `bool`)
  - **功能**: 无事务时数据IO的默认输出电平
  - **取值**: true(高电平) 或 false(低电平)
  - **注意**: ESP32不支持此功能

- **`max_transfer_sz`** (类型: `int`)
  - **功能**: 单次传输的最大字节数
  - **默认值**: 启用DMA时为4092，禁用DMA时为SOC_SPI_MAXIMUM_BUFFER_SIZE
  - **限制**: 受DMA描述符和内存限制

- **`flags`** (类型: `uint32_t`)
  - **功能**: 总线能力和配置标志
  - **可用标志**: 
    - `SPICOMMON_BUSFLAG_MASTER`: 主机模式
    - `SPICOMMON_BUSFLAG_SLAVE`: 从机模式
    - `SPICOMMON_BUSFLAG_IOMUX_PINS`: 使用IOMUX引脚
    - `SPICOMMON_BUSFLAG_GPIO_PINS`: 强制使用GPIO矩阵
    - `SPICOMMON_BUSFLAG_DUAL`: 双线模式
    - `SPICOMMON_BUSFLAG_QUAD`: 四线模式
    - `SPICOMMON_BUSFLAG_OCTAL`: 八线模式
    - `SPICOMMON_BUSFLAG_SLP_ALLOW_PD`: 允许轻度睡眠时断电

- **`isr_cpu_id`** (类型: `esp_intr_cpu_affinity_t`)
  - **功能**: 选择执行SPI ISR的CPU核心
  - **作用**: 绑定ISR到特定核心以优化性能

- **`intr_flags`** (类型: `int`)
  - **功能**: 中断分配标志
  - **用途**: 设置中断优先级和IRAM属性
  - **限制**: EDGE、INTRDISABLED属性会被驱动忽略

### spi_common_dma_t

SPI DMA通道枚举类型：

```c
typedef enum {
    SPI_DMA_DISABLED = 0,     ///< 不为SPI启用DMA
#if CONFIG_IDF_TARGET_ESP32
    SPI_DMA_CH1      = 1,     ///< 启用DMA，选择DMA通道1
    SPI_DMA_CH2      = 2,     ///< 启用DMA，选择DMA通道2
#endif
    SPI_DMA_CH_AUTO  = 3,     ///< 启用DMA，通道由驱动自动选择
} spi_common_dma_t;
```

**详细说明：**

- **`SPI_DMA_DISABLED`**
  - **含义**: 禁用DMA传输
  - **限制**: 传输大小受限于内部缓冲区
  - **适用**: 小数据量传输或仅SPI Flash使用总线时

- **`SPI_DMA_CH1` / `SPI_DMA_CH2`** (仅ESP32)
  - **含义**: 手动选择特定DMA通道
  - **用途**: 需要特定通道资源时
  - **注意**: 仅ESP32支持多个独立DMA通道

- **`SPI_DMA_CH_AUTO`**
  - **含义**: 自动分配DMA通道
  - **推荐**: 大多数情况下的最佳选择
  - **优势**: 驱动自动管理DMA资源

## 1. SPI总线初始化和释放函数

### spi_bus_initialize
**函数原型：**
```c
esp_err_t spi_bus_initialize(spi_host_device_t host_id, const spi_bus_config_t *bus_config, spi_dma_chan_t dma_chan);
```
**参数：**
host_id：控制此总线的SPI外设(SPI1_HOST、SPI2_HOST或SPI3_HOST)
bus_config：指向spi_bus_config_t结构体的指针，指定主机应如何初始化
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
**作用：**初始化SPI总线，配置GPIO引脚和DMA通道。如果选择DMA通道，任何发送和接收缓冲区都应分配在DMA能访问的内存中

**重要说明：**
- SPI0/1不支持此函数
- SPI的ISR总是在调用此函数的核心上执行
- 不要在该核心上使ISR饿死，否则SPI事务将无法处理
- 使用DMA时，缓冲区必须分配在DMA可访问的内存中

### spi_bus_free
**函数原型：**
```c
esp_err_t spi_bus_free(spi_host_device_t host_id);
```
**参数：**
host_id：要释放的SPI外设
**返回值：**
ESP_OK：释放成功
ESP_ERR_INVALID_ARG：参数无效
ESP_ERR_INVALID_STATE：总线未初始化或总线上的设备未全部释放
**作用：**释放SPI总线。为了成功释放，必须先移除所有设备

**注意事项：**
- 必须先调用spi_bus_remove_device()移除所有设备
- 释放后GPIO引脚会被重置
- DMA资源会被释放

## 2. SPI内存分配函数

### spi_bus_dma_memory_alloc
**函数原型：**
```c
void *spi_bus_dma_memory_alloc(spi_host_device_t host_id, size_t size, uint32_t extra_heap_caps);
```
**参数：**
host_id：将使用该内存的SPI外设
size：要分配的内存大小(字节)
extra_heap_caps：基于MALLOC_CAP_DMA的额外堆能力
**返回值：**
成功时返回内存指针
失败时返回NULL
**作用：**为SPI驱动分配DMA可访问的内存的辅助函数。该API会自动处理缓存和硬件对齐。使用free()释放由此函数分配的内存

**使用场景：**
- 需要DMA传输的发送/接收缓冲区
- 自动处理对齐要求
- 确保内存在DMA可访问区域

**注意事项：**
- 不支持外部存储器(SPIRAM)
- 自动处理缓存对齐
- 使用标准free()函数释放
