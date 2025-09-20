# ESP32 UART参考


## 1. UART驱动初始化

### uart_param_config

**函数原型：**

```c
esp_err_t uart_param_config(uart_port_t uart_num, const uart_config_t *uart_config);
```

**参数：**

- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- uart_config：UART参数设置结构体指针

**uart_config_t结构体：**

```c
typedef struct {
    int baud_rate;                      // UART波特率
    uart_word_length_t data_bits;       // UART字节大小
    uart_parity_t parity;               // UART奇偶校验模式
    uart_stop_bits_t stop_bits;         // UART停止位
    uart_hw_flowcontrol_t flow_ctrl;    // UART硬件流控制模式
    uint8_t rx_flow_ctrl_thresh;        // UART硬件RTS阈值
    union {
        uart_sclk_t source_clk;         // UART源时钟选择
        lp_uart_sclk_t lp_source_clk;   // LP_UART源时钟选择
    };
    struct {
        uint32_t allow_pd: 1;           // 允许在系统进入睡眠模式时关闭电源域
        uint32_t backup_before_sleep: 1; // 已弃用，与allow_pd含义相同
    } flags;
} uart_config_t;
```

**返回值：**

- ESP_OK：成功
- ESP_FAIL：参数错误，如波特率不可达

**作用：**设置UART配置参数

### uart_set_pin

**函数原型：**

```c
esp_err_t uart_set_pin(uart_port_t uart_num, int tx_io_num, int rx_io_num, int rts_io_num, int cts_io_num);
```

**参数：**

- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- tx_io_num：UART TX引脚GPIO编号
- rx_io_num：UART RX引脚GPIO编号
- rts_io_num：UART RTS引脚GPIO编号
- cts_io_num：UART CTS引脚GPIO编号

**返回值：**

- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**为UART外设分配信号到GPIO引脚

### uart_driver_install

**函数原型：**
```c
esp_err_t uart_driver_install(uart_port_t uart_num, int rx_buffer_size, int tx_buffer_size, int queue_size, QueueHandle_t* uart_queue, int intr_alloc_flags);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- rx_buffer_size：UART接收环形缓冲区大小，应大于UART_HW_FIFO_LEN(uart_num)
- tx_buffer_size：UART发送环形缓冲区大小，应大于0或大于UART_HW_FIFO_LEN(uart_num)
- queue_size：UART事件队列大小/深度
- uart_queue：UART事件队列句柄（输出参数），可为NULL
- intr_alloc_flags：中断分配标志位，使用ESP_INTR_FLAG_*值的组合

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**安装UART驱动并将UART设置为默认配置，UART ISR处理程序将附加到运行此函数的CPU核心

### uart_driver_delete
**函数原型：**
```c
esp_err_t uart_driver_delete(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**卸载UART驱动

### uart_is_driver_installed
**函数原型：**

```c
bool uart_is_driver_installed(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- true：驱动已安装
- false：驱动未安装

**作用：**检查驱动是否已安装

## 2. UART参数设置和读取*

### uart_set_baudrate
**函数原型：**
```c
esp_err_t uart_set_baudrate(uart_port_t uart_num, uint32_t baudrate);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- baudrate：UART波特率

**返回值：**
- ESP_FAIL：参数错误，如波特率不可达
- ESP_OK：成功

**作用：**设置期望的UART波特率，实际设置的波特率可能因舍入误差与用户配置值略有偏差

### uart_get_baudrate
**函数原型：**
```c
esp_err_t uart_get_baudrate(uart_port_t uart_num, uint32_t* baudrate);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- baudrate：UART波特率指针，用于接收返回值

**返回值：**
- ESP_FAIL：参数错误
- ESP_OK：成功，结果将放入(*baudrate)

**作用：**获取实际的UART波特率

### uart_get_sclk_freq
**函数原型：**
```c
esp_err_t uart_get_sclk_freq(uart_sclk_t sclk, uint32_t* out_freq_hz);
```
**参数：**
- sclk：时钟源
- out_freq_hz：输出频率，单位为Hz

**返回值：**
- ESP_ERR_INVALID_ARG：时钟源不支持
- ESP_OK：成功

**作用：**获取HP UART端口的时钟源频率

### uart_set_word_length
**函数原型：**
```c
esp_err_t uart_set_word_length(uart_port_t uart_num, uart_word_length_t data_bit);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- data_bit：UART数据位

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置UART数据位

### uart_get_word_length
**函数原型：**
```c
esp_err_t uart_get_word_length(uart_port_t uart_num, uart_word_length_t* data_bit);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- data_bit：UART数据位指针，用于接收返回值

**返回值：**
- ESP_FAIL：参数错误
- ESP_OK：成功，结果将放入(*data_bit)

**作用：**获取UART数据位配置

### uart_set_stop_bits
**函数原型：**
```c
esp_err_t uart_set_stop_bits(uart_port_t uart_num, uart_stop_bits_t stop_bits);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- stop_bits：UART停止位

**返回值：**
- ESP_OK：成功
- ESP_FAIL：失败

**作用：**设置UART停止位

### uart_get_stop_bits
**函数原型：**
```c
esp_err_t uart_get_stop_bits(uart_port_t uart_num, uart_stop_bits_t* stop_bits);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- stop_bits：UART停止位指针，用于接收返回值

**返回值：**
- ESP_FAIL：参数错误
- ESP_OK：成功，结果将放入(*stop_bit)

**作用：**获取UART停止位配置

### uart_set_parity
**函数原型：**
```c
esp_err_t uart_set_parity(uart_port_t uart_num, uart_parity_t parity_mode);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- parity_mode：UART奇偶校验配置枚举

**返回值：**
- ESP_FAIL：参数错误
- ESP_OK：成功

**作用：**设置UART奇偶校验模式

### uart_get_parity
**函数原型：**
```c
esp_err_t uart_get_parity(uart_port_t uart_num, uart_parity_t* parity_mode);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- parity_mode：UART奇偶校验模式指针，用于接收返回值

**返回值：**
- ESP_FAIL：参数错误
- ESP_OK：成功，结果将放入(*parity_mode)

**作用：**获取UART奇偶校验模式配置

### uart_set_hw_flow_ctrl
**函数原型：**
```c
esp_err_t uart_set_hw_flow_ctrl(uart_port_t uart_num, uart_hw_flowcontrol_t flow_ctrl, uint8_t rx_thresh);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- flow_ctrl：硬件流控制模式
- rx_thresh：硬件RX流控制阈值(0 ~ UART_HW_FIFO_LEN(uart_num))，仅当设置UART_HW_FLOWCTRL_RTS时有效

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置硬件流控制

### uart_get_hw_flow_ctrl
**函数原型：**
```c
esp_err_t uart_get_hw_flow_ctrl(uart_port_t uart_num, uart_hw_flowcontrol_t* flow_ctrl);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- flow_ctrl：不同流控制模式的选项指针

**返回值：**
- ESP_FAIL：参数错误
- ESP_OK：成功，结果将放入(*flow_ctrl)

**作用：**获取UART硬件流控制配置

### uart_set_sw_flow_ctrl
**函数原型：**

```c
esp_err_t uart_set_sw_flow_ctrl(uart_port_t uart_num, bool enable, uint8_t rx_thresh_xon, uint8_t rx_thresh_xoff);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- enable：开关控制
- rx_thresh_xon：低水位标记
- rx_thresh_xoff：高水位标记

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置软件流控制。接收方缓冲区快满了 → 发送 XOFF 字符（ASCII 0x13，即 DC3）给发送方 发送方收到 XOFF → 暂停发送 接收方处理完数据，缓冲区空了 → 发送 XON 字符（ASCII 0x11，即 DC1） 发送方收到 XON → 恢复发送

### uart_set_rts

**函数原型：**

```c
esp_err_t uart_set_rts(uart_port_t uart_num, int level);
```

**参数：**

- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- level：1表示RTS输出低电平（有效），0表示RTS输出高电平（阻塞）

**返回值：**

- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**手动设置UART RTS引脚电平，UART必须配置为禁用硬件流控制

### uart_set_dtr

**函数原型：**

```c
esp_err_t uart_set_dtr(uart_port_t uart_num, int level);
```

**参数：**

- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- level：1表示DTR输出低电平，0表示DTR输出高电平

**返回值：**

- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**手动设置UART DTR引脚电平，UART必须配置为禁用硬件流控制

### uart_set_line_inverse

**函数原型：**

```c
esp_err_t uart_set_line_inverse(uart_port_t uart_num, uint32_t inverse_mask);
```

**参数：**

- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- inverse_mask：选择需要反转的线路，使用uart_signal_inv_t的ORred掩码

**返回值：**

- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**这个函数用于 **反转 UART 信号的逻辑电平**，也就是说：

- 默认情况下：
  - 逻辑 `1`（空闲/停止位）→ 高电平（3.3V）
  - 逻辑 `0`（起始位/数据位）→ 低电平（0V）
- 启用反转后：
  - 逻辑 `1` → 变成低电平
  - 逻辑 `0` → 变成高电平

## 3. UART数据发送

### uart_write_bytes
**函数原型：**
```c
int uart_write_bytes(uart_port_t uart_num, const void* src, size_t size);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- src：数据缓冲区地址
- size：要发送的数据长度

**返回值：**
- (-1)：参数错误
- 其他值(>=0)：推送到TX FIFO的字节数

**作用：**将数据从给定缓冲区发送到UART端口

### uart_write_bytes_with_break
**函数原型：**
```c
int uart_write_bytes_with_break(uart_port_t uart_num, const void* src, size_t size, int brk_len);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- src：数据缓冲区地址
- size：要发送的数据长度
- brk_len：中断信号持续时间（单位：当前波特率下发送一位所需的时间）

**返回值：**
- (-1)：参数错误
- 其他值(>=0)：推送到TX FIFO的字节数

**作用：**发送数据后发送中断信号。传输结束时会添加串行中断信号。“串行中断信号”意味着 Tx 线保持低电平的时间长于一个数据帧。

### uart_tx_chars
**函数原型：**
```c
int uart_tx_chars(uart_port_t uart_num, const char* buffer, uint32_t len);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- buffer：数据缓冲区地址
- len：要发送的数据长度

**返回值：**
- (-1)：参数错误
- 其他值(>=0)：推送到TX FIFO的字节数

**作用：**将数据发送到UART端口，不会等待TX FIFO有足够空间，仅填充可用的TX FIFO

### uart_wait_tx_done
**函数原型：**
```c
esp_err_t uart_wait_tx_done(uart_port_t uart_num, TickType_t ticks_to_wait);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- ticks_to_wait：超时时间，以RTOS时钟节拍计数

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误
- ESP_ERR_TIMEOUT：超时

**作用：**等待直到UART TX FIFO为空

### uart_set_tx_idle_num
**函数原型：**
```c
esp_err_t uart_set_tx_idle_num(uart_port_t uart_num, uint16_t idle_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- idle_num：TX FIFO为空后的空闲间隔（单位：当前波特率下发送一位所需的时间）

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置TX FIFO为空后的UART空闲间隔

## 4. UART数据接收

### uart_read_bytes
**函数原型：**
```c
int uart_read_bytes(uart_port_t uart_num, void* buf, uint32_t length, TickType_t ticks_to_wait);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- buf：数据缓冲区指针
- length：要读取的数据长度
- ticks_to_wait：超时时间，以RTOS时钟节拍计数

**返回值：**
- (-1)：错误
- 其他值(>=0)：从UART FIFO读取的字节数

**作用：**从UART缓冲区读取字节

### uart_get_buffered_data_len
**函数原型：**
```c
esp_err_t uart_get_buffered_data_len(uart_port_t uart_num, size_t* size);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- size：用于接收缓冲区数据长度的size_t指针

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**获取UART缓冲区中的数据长度

### uart_get_tx_buffer_free_size
**函数原型：**
```c
esp_err_t uart_get_tx_buffer_free_size(uart_port_t uart_num, size_t *size);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- size：用于接收空闲空间大小的size_t指针

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误

**作用：**获取UART TX环形缓冲区空闲空间大小

### uart_flush_input
**函数原型：**
```c
esp_err_t uart_flush_input(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**清空UART输入缓冲区，注意是buffer中的数据，不是硬件FIFO中的数据

### uart_flush
**函数原型：**

```c
esp_err_t uart_flush(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**清空UART输出缓冲区（TX FIFO），等待所有数据传输完成

## 5. UART中断配置

### uart_intr_config
**函数原型：**
```c
esp_err_t uart_intr_config(uart_port_t uart_num, const uart_intr_config_t *intr_conf);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- intr_conf：UART中断设置结构体指针

**uart_intr_config_t结构体：**
```c
typedef struct {
    uint32_t intr_enable_mask;          // UART中断使能掩码
    uint8_t  rx_timeout_thresh;         // UART超时中断阈值（单位：发送一个字节的时间）
    uint8_t  txfifo_empty_intr_thresh;  // UART TX空中断阈值
    uint8_t  rxfifo_full_thresh;        // UART RX满中断阈值
} uart_intr_config_t;
```

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**配置UART中断

### uart_enable_intr_mask
**函数原型：**
```c
esp_err_t uart_enable_intr_mask(uart_port_t uart_num, uint32_t enable_mask);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- enable_mask：使能位的位掩码

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置UART中断使能

### uart_disable_intr_mask
**函数原型：**
```c
esp_err_t uart_disable_intr_mask(uart_port_t uart_num, uint32_t disable_mask);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- disable_mask：禁用位的位掩码

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**清除UART中断使能位

### uart_clear_intr_status
**函数原型：**
```c
esp_err_t uart_clear_intr_status(uart_port_t uart_num, uint32_t clr_mask);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- clr_mask：要清除的中断状态的位掩码

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**清除UART中断状态

### uart_enable_rx_intr
**函数原型：**
```c
esp_err_t uart_enable_rx_intr(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**使能UART RX中断（RX_FULL & RX_TIMEOUT中断）

### uart_disable_rx_intr
**函数原型：**
```c
esp_err_t uart_disable_rx_intr(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**禁用UART RX中断（RX_FULL & RX_TIMEOUT中断）

### uart_enable_tx_intr
**函数原型：**
```c
esp_err_t uart_enable_tx_intr(uart_port_t uart_num, int enable, int thresh);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- enable：1表示使能，0表示禁用
- thresh：TX中断阈值，0 ~ UART_HW_FIFO_LEN(uart_num)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**使能UART TX中断（TX_FULL & TX_TIMEOUT中断）

### uart_disable_tx_intr
**函数原型：**
```c
esp_err_t uart_disable_tx_intr(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**禁用UART TX中断（TX_FULL & TX_TIMEOUT中断）

## 12. UART模式检测

### uart_enable_pattern_det_baud_intr
**函数原型：**
```c
esp_err_t uart_enable_pattern_det_baud_intr(uart_port_t uart_num, char pattern_chr, uint8_t chr_num, int chr_tout, int post_idle, int pre_idle);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- pattern_chr：用于检测的模式字符
- chr_num：用于检测的字符数量
- chr_tout：字符超时时间
- post_idle：后置空闲时间
- pre_idle：前置空闲时间

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**启用模式检测功能

### uart_disable_pattern_det_intr
**函数原型：**
```c
esp_err_t uart_disable_pattern_det_intr(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**禁用模式检测功能

### uart_pattern_get_pos
**函数原型：**
```c
int uart_pattern_get_pos(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- (-1)：错误
- 其他值(>=0)：模式位置索引

**作用：**获取模式检测的位置索引

### uart_pattern_pop_pos
**函数原型：**
```c
int uart_pattern_pop_pos(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- (-1)：错误
- 其他值(>=0)：模式位置索引

**作用：**从模式检测队列中弹出一个模式位置

### uart_pattern_queue_reset
**函数原型：**
```c
esp_err_t uart_pattern_queue_reset(uart_port_t uart_num, int queue_length);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- queue_length：模式检测队列长度

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**重置模式检测队列

### uart_set_mode
**函数原型：**
```c
esp_err_t uart_set_mode(uart_port_t uart_num, uart_mode_t mode);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- mode：UART模式

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置UART通信模式

## 11. UART阈值配置

### uart_set_rx_full_threshold
**函数原型：**
```c
esp_err_t uart_set_rx_full_threshold(uart_port_t uart_num, int threshold);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- threshold：RX FIFO满阈值

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置RX FIFO满阈值

### uart_set_tx_empty_threshold
**函数原型：**
```c
esp_err_t uart_set_tx_empty_threshold(uart_port_t uart_num, int threshold);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- threshold：TX FIFO空阈值

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置TX FIFO空阈值

### uart_set_rx_timeout
**函数原型：**
```c
esp_err_t uart_set_rx_timeout(uart_port_t uart_num, uint8_t tout_thresh);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- tout_thresh：RX超时阈值

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**设置RX超时阈值

## 13. UART冲突检测

### uart_get_collision_flag
**函数原型：**
```c
esp_err_t uart_get_collision_flag(uart_port_t uart_num, bool *collision_flag);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- collision_flag：用于接收冲突标志的bool指针

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**获取UART冲突检测标志

## 14. UART自动波特率检测

### uart_detect_baudrate_start
**函数原型：**
```c
esp_err_t uart_detect_baudrate_start(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**启动自动波特率检测

### uart_detect_baudrate_stop
**函数原型：**
```c
esp_err_t uart_detect_baudrate_stop(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**停止自动波特率检测

### uart_detect_baudrate_result
**函数原型：**
```c
esp_err_t uart_detect_baudrate_result(uart_port_t uart_num, uint32_t *baudrate);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- baudrate：用于接收检测到的波特率的uint32_t指针

**返回值：**
- ESP_OK：成功
- ESP_FAIL：参数错误

**作用：**获取自动波特率检测结果

### uart_detect_bitrate_start
**函数原型：**
```c
esp_err_t uart_detect_bitrate_start(uart_port_t uart_num, const uart_bitrate_detect_config_t *config);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- config：UART比特率检测配置参数指针

**uart_bitrate_detect_config_t结构体：**
```c
typedef struct {
    int rx_io_num;          /*!< GPIO引脚号用于接收信号 */
    uart_sclk_t source_clk; /*!< 时钟源频率，越高检测越精确 */
} uart_bitrate_detect_config_t;
```

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误
- ESP_FAIL：无空闲UART端口或source_clk无效

**作用：**启动UART比特率检测（自动波特率检测），用于检测输入数据信号的比特率

### uart_detect_bitrate_stop
**函数原型：**
```c
esp_err_t uart_detect_bitrate_stop(uart_port_t uart_num, bool deinit, uart_bitrate_res_t *ret_res);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- deinit：是否在完成测量后释放UART端口
- ret_res：用于存储测量结果的结构体指针

**uart_bitrate_res_t结构体：**
```c
typedef struct {
    uint32_t low_period;    /*!< 存储低电平脉冲的最小时钟计数 */
    uint32_t high_period;   /*!< 存储高电平脉冲的最小时钟计数 */
    uint32_t pos_period;    /*!< 存储两个正边沿之间的最小时钟计数 */
    uint32_t neg_period;    /*!< 存储两个负边沿之间的最小时钟计数 */
    uint32_t edge_cnt;      /*!< RX边沿变化的计数 */
    uint32_t clk_freq_hz;   /*!< 用于计数测量结果的时钟频率，单位Hz */
} uart_bitrate_res_t;
```

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误
- ESP_FAIL：未知时钟频率

**作用：**停止比特率检测并获取测量结果

## 16. UART高级功能

### uart_wait_tx_idle_polling
**函数原型：**
```c
esp_err_t uart_wait_tx_idle_polling(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误
- ESP_FAIL：驱动未安装

**作用：**等待UART TX空闲（轮询模式），等待所有数据传输完成

### uart_set_loop_back
**函数原型：**
```c
esp_err_t uart_set_loop_back(uart_port_t uart_num, bool loop_back_en);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- loop_back_en：是否启用回环模式（true：启用，false：禁用）

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误
- ESP_FAIL：驱动未安装

**作用：**配置TX信号回环到RX模块，仅用于测试用途

### uart_set_always_rx_timeout
**函数原型：**
```c
void uart_set_always_rx_timeout(uart_port_t uart_num, bool always_rx_timeout_en);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- always_rx_timeout_en：是否始终启用RX超时（true：即使FIFO满也触发超时中断，false：默认行为）

**作用：**配置UART RX超时中断的行为，即使FIFO满也触发超时中断

## 10. UART事件处理

### uart_event_type_t枚举
```c
typedef enum {
    UART_DATA,              // 接收器接收字节超时或接收数据超过rxfifo_full_thresh指定值时触发
    UART_BREAK,             // 接收器检测到NULL字符时触发
    UART_BUFFER_FULL,       // RX环形缓冲区满时触发
    UART_FIFO_OVF,          // 接收数据超过RX FIFO容量时触发
    UART_FRAME_ERR,         // 接收器检测到数据帧错误时触发
    UART_PARITY_ERR,        // 检测到接收数据的奇偶校验错误时触发
    UART_DATA_BREAK,        // 数据传输后发送中断信号的内部事件
    UART_PATTERN_DET,       // 检测到传入数据中指定模式时触发
    UART_WAKEUP,            // 检测到唤醒信号时触发（SOC_UART_SUPPORT_WAKEUP_INT支持时）
    UART_EVENT_MAX,         // UART事件的最大索引
} uart_event_type_t;
```

### uart_event_t结构体
```c
typedef struct {
    uart_event_type_t type; // UART事件类型
    size_t size;            // UART_DATA事件的数据大小
    bool timeout_flag;      // UART_DATA事件的UART数据读取超时标志
} uart_event_t;
```

## 17. UART VFS集成

### uart_vfs_dev_register
**函数原型：**
```c
void uart_vfs_dev_register(void);
```
**作用：**添加/dev/uart虚拟文件系统驱动，从启动代码调用以启用串行输出

### uart_vfs_dev_port_set_rx_line_endings
**函数原型：**
```c
int uart_vfs_dev_port_set_rx_line_endings(int uart_num, esp_line_endings_t mode);
```
**参数：**
- uart_num：UART编号
- mode：发送到UART的行结束符
  - ESP_LINE_ENDINGS_CRLF：将CRLF转换为LF
  - ESP_LINE_ENDINGS_CR：将CR转换为LF
  - ESP_LINE_ENDINGS_LF：不修改

**返回值：**
- 0：成功
- -1：错误

**作用：**设置指定UART上期望接收的行结束符

### uart_vfs_dev_port_set_tx_line_endings
**函数原型：**
```c
int uart_vfs_dev_port_set_tx_line_endings(int uart_num, esp_line_endings_t mode);
```
**参数：**
- uart_num：UART编号
- mode：发送到UART的行结束符
  - ESP_LINE_ENDINGS_CRLF：将LF转换为CRLF
  - ESP_LINE_ENDINGS_CR：将LF转换为CR
  - ESP_LINE_ENDINGS_LF：不修改

**返回值：**
- 0：成功
- -1：错误

**作用：**设置要发送到指定UART的行结束符

### uart_vfs_dev_use_nonblocking
**函数原型：**
```c
void uart_vfs_dev_use_nonblocking(int uart_num);
```
**参数：**
- uart_num：UART外设编号

**作用：**设置VFS使用简单的读写UART函数，读取为非阻塞，写入为忙等待直到TX FIFO有足够空间，这些函数默认使用

### uart_vfs_dev_use_driver
**函数原型：**
```c
void uart_vfs_dev_use_driver(int uart_num);
```
**参数：**
- uart_num：UART外设编号

**作用：**设置VFS使用UART驱动进行读写，应用程序在调用这些函数前必须配置UART驱动，读写为阻塞和中断驱动

## 15. UART唤醒功能

### uart_wakeup_setup
**函数原型：**
```c
esp_err_t uart_wakeup_setup(uart_port_t uart_num, const uart_wakeup_cfg_t *cfg);
```
**参数：**
- uart_num：要初始化唤醒的UART端口（如UART_NUM_0, UART_NUM_1等）
- cfg：包含唤醒配置设置的uart_wakeup_cfg_t结构体指针

**uart_wakeup_cfg_t结构体：**
```c
typedef struct {
    uart_wakeup_mode_t wakeup_mode;     // 唤醒模式选择
#if SOC_UART_WAKEUP_SUPPORT_ACTIVE_THRESH_MODE
    uint16_t rx_edge_threshold;         // 活动阈值唤醒模式下使用的RX边沿变化阈值
#endif
#if SOC_UART_WAKEUP_SUPPORT_FIFO_THRESH_MODE
    uint16_t rx_fifo_threshold;         // RX FIFO阈值唤醒模式下使用的接收数据字节数
#endif
#if SOC_UART_WAKEUP_SUPPORT_CHAR_SEQ_MODE
    char wake_chars_seq[SOC_UART_WAKEUP_CHARS_SEQ_MAX_LEN]; // 字符序列检测唤醒模式下使用的字符序列
#endif
} uart_wakeup_cfg_t;
```

**返回值：**
- ESP_OK：唤醒配置成功应用
- ESP_ERR_INVALID_ARG：提供的配置无效

**作用：**初始化UART唤醒功能

### uart_wakeup_clear
**函数原型：**
```c
esp_err_t uart_wakeup_clear(uart_port_t uart_num, uart_wakeup_mode_t wakeup_mode);
```
**参数：**
- uart_num：要初始化唤醒的UART端口（如UART_NUM_0, UART_NUM_1等）
- wakeup_mode：在uart_wakeup_setup中设置的UART唤醒模式

**返回值：**
- ESP_OK：清除唤醒配置成功

**作用：**清除UART唤醒配置

### uart_set_wakeup_threshold
**函数原型：**
```c
esp_err_t uart_set_wakeup_threshold(uart_port_t uart_num, int wakeup_threshold);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- wakeup_threshold：用于轻度睡眠唤醒的RX边沿数量，值为3-0x3ff

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误或wakeup_threshold超出[3, 0x3ff]范围

**作用：**设置用于轻度睡眠唤醒的RX引脚信号边沿数量阈值

**说明：**
UART可用于从轻度睡眠中唤醒系统。该功能通过计算RX引脚上的正边沿数量并与阈值进行比较来工作。当计数超过阈值时，系统从轻度睡眠中唤醒。
停止位和奇偶校验位（如果启用）也会影响边沿数量。例如，ASCII码为97的字母'a'在导线上编码为0100001101（使用8n1配置），包括起始位和停止位。此序列有3个正边沿（从0到1的转换）。因此，要在发送'a'时唤醒系统，设置wakeup_threshold=3。
触发唤醒的字符不会被UART接收（即无法从UART FIFO中获取）。根据波特率，之后的几个字符也可能不会被接收。

### uart_get_wakeup_threshold
**函数原型：**
```c
esp_err_t uart_get_wakeup_threshold(uart_port_t uart_num, int* out_wakeup_threshold);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- out_wakeup_threshold：用于接收唤醒阈值的int指针

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误

**作用：**获取UART唤醒阈值

### uart_wait_tx_idle_polling
**函数原型：**
```c
esp_err_t uart_wait_tx_idle_polling(uart_port_t uart_num);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误
- ESP_FAIL：驱动未安装

**作用：**等待UART TX空闲（轮询模式），等待所有数据传输完成

### uart_set_loop_back
**函数原型：**
```c
esp_err_t uart_set_loop_back(uart_port_t uart_num, bool loop_back_en);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- loop_back_en：是否启用回环模式（true：启用，false：禁用）

**返回值：**
- ESP_OK：成功
- ESP_ERR_INVALID_ARG：参数错误
- ESP_FAIL：驱动未安装

**作用：**配置TX信号回环到RX模块，仅用于测试用途

### uart_set_always_rx_timeout
**函数原型：**
```c
void uart_set_always_rx_timeout(uart_port_t uart_num, bool always_rx_timeout_en);
```
**参数：**
- uart_num：UART端口号，最大端口号为(UART_NUM_MAX -1)
- always_rx_timeout_en：是否始终启用RX超时（true：即使FIFO满也触发超时中断，false：默认行为）

**作用：**配置UART RX超时中断的行为，即使FIFO满也触发超时中断

## 18. UART选择通知

### uart_set_select_notif_callback
**函数原型：**
```c
void uart_set_select_notif_callback(uart_port_t uart_num, uart_select_notif_callback_t uart_select_notif_callback);
```
**参数：**
- uart_num：UART端口号
- uart_select_notif_callback：回调函数

**作用：**为select()事件设置通知回调函数

### uart_get_selectlock
**函数原型：**
```c
portMUX_TYPE *uart_get_selectlock(void);
```
**返回值：**
- 保护select()通知的互斥锁指针

**作用：**获取保护select()通知的互斥锁

## 19. UHCI高级功能

### uhci_new_controller
**函数原型：**
```c
esp_err_t uhci_new_controller(const uhci_controller_config_t *config, uhci_controller_handle_t *ret_uhci_ctrl);
```
**参数：**
- config：指向包含UHCI控制器配置参数的uhci_controller_config_t结构体的指针
- ret_uhci_ctrl：指向变量的指针，用于存储新创建的UHCI控制器的句柄

**uhci_controller_config_t结构体：**
```c
typedef struct {
    uart_port_t uart_port;                    // 连接到UHCI控制器的UART端口
    size_t tx_trans_queue_depth;              // 内部传输队列深度
    size_t max_transmit_size;                 // 单次事务中的最大传输大小（字节）
    size_t max_receive_internal_mem;        // 内部DMA使用内存
    size_t dma_burst_size;                    // DMA突发大小（字节）
    size_t max_packet_receive;              // 最大接收大小
    struct {
        uint16_t rx_brk_eof: 1;               // 接收到NULL帧时结束有效载荷接收
        uint16_t idle_eof: 1;                 // UART空闲时结束有效载荷接收
        uint16_t length_eof: 1;               // 接收字节计数达到特定值时结束有效载荷接收
    } rx_eof_flags;                           // UHCI eof标志
} uhci_controller_config_t;
```

**返回值：**
- ESP_OK：控制器成功创建和初始化
- ESP_ERR_INVALID_ARG：一个或多个参数无效
- ESP_ERR_NO_MEM：控制器内存分配失败
- 其他错误代码：底层硬件或驱动初始化失败

**作用：**创建并初始化新的UHCI控制器

### uhci_del_controller
**函数原型：**
```c
esp_err_t uhci_del_controller(uhci_controller_handle_t uhci_ctrl);
```
**参数：**
- uhci_ctrl：先前使用uhci_new_controller()创建的UHCI控制器的句柄

**返回值：**
- ESP_OK：UHCI驱动成功卸载，资源已释放
- ESP_ERR_INVALID_ARG：提供的uhci_ctrl句柄无效或为空

**作用：**卸载UHCI驱动并释放资源

### uhci_transmit
**函数原型：**
```c
esp_err_t uhci_transmit(uhci_controller_handle_t uhci_ctrl, uint8_t *write_buffer, size_t write_size);
```
**参数：**
- uhci_ctrl：先前使用uhci_new_controller()创建的UHCI控制器的句柄
- write_buffer：包含要发送数据的缓冲区的指针
- write_size：要从缓冲区发送的字节数

**返回值：**
- ESP_OK：数据成功排队等待传输
- ESP_ERR_INVALID_ARG：无效参数

**作用：**使用UHCI控制器发送数据

### uhci_receive
**函数原型：**
```c
esp_err_t uhci_receive(uhci_controller_handle_t uhci_ctrl, uint8_t *read_buffer, size_t buffer_size);
```
**参数：**
- uhci_ctrl：先前使用uhci_new_controller()创建的UHCI控制器的句柄
- read_buffer：用于存储接收数据的缓冲区的指针
- buffer_size：读取缓冲区的大小

**返回值：**
- ESP_OK：数据成功接收并写入缓冲区
- ESP_ERR_INVALID_ARG：无效参数

**作用：**从UHCI控制器接收数据

### uhci_register_event_callbacks
**函数原型：**
```c
esp_err_t uhci_register_event_callbacks(uhci_controller_handle_t uhci_ctrl, const uhci_event_callbacks_t *cbs, void *user_data);
```
**参数：**
- uhci_ctrl：先前使用uhci_new_controller()创建的UHCI控制器的句柄
- cbs：指向定义回调函数的uhci_event_callbacks_t结构体的指针
- user_data：用户定义的数据指针，将在调用回调函数时传递

**uhci_event_callbacks_t结构体：**
```c
typedef struct {
    uhci_rx_event_callback_t on_rx_trans_event;   // 处理接收完成的回调函数
    uhci_tx_done_callback_t on_tx_trans_done;   // 处理传输完成的回调函数
} uhci_event_callbacks_t;
```

**返回值：**
- ESP_OK：事件回调成功注册
- ESP_ERR_INVALID_ARG：无效参数

**作用：**为UHCI控制器注册事件回调函数

### uhci_wait_all_tx_transaction_done
**函数原型：**
```c
esp_err_t uhci_wait_all_tx_transaction_done(uhci_controller_handle_t uhci_ctrl, int timeout_ms);
```
**参数：**
- uhci_ctrl：由uhci_new_controller创建的UHCI控制器
- timeout_ms：超时时间（毫秒），-1表示永远等待

**返回值：**
- ESP_OK：所有待处理的TX事务完成并已回收
- ESP_ERR_INVALID_ARG：由于无效参数导致等待失败
- ESP_ERR_TIMEOUT：等待超时
- ESP_FAIL：由于其他错误导致等待失败

**作用：**等待所有待处理的TX事务完成