# ESP32 LOG 日志系统使用指南

ESP-IDF 提供了一个完整的日志系统，支持多种日志级别和灵活的配置选项。

## 1. 日志级别

ESP-IDF 定义了以下日志级别（按严重程度递减）：

| 级别 | 宏定义 | 描述 |
|------|--------|------|
| 错误 | `ESP_LOGE` | 错误条件，系统无法正常工作 |
| 警告 | `ESP_LOGW` | 警告条件，可能导致问题 |
| 信息 | `ESP_LOGI` | 一般信息消息 |
| 调试 | `ESP_LOGD` | 调试信息 |
| 详细 | `ESP_LOGV` | 详细调试信息 |

## 2. 基本用法

### 2.1 包含头文件

```c
#include "esp_log.h"
```

### 2.2 定义标签

```c
static const char *TAG = "MY_MODULE";
```

### 2.3 使用日志宏

```c
void example_function(void)
{
    ESP_LOGE(TAG, "这是一个错误消息");
    ESP_LOGW(TAG, "这是一个警告消息");
    ESP_LOGI(TAG, "这是一个信息消息");
    ESP_LOGD(TAG, "这是一个调试消息");
    ESP_LOGV(TAG, "这是一个详细消息");
}
```

### 2.4 格式化输出

```c
int temperature = 25;
float voltage = 3.3;
char device_name[] = "ESP32";

ESP_LOGI(TAG, "设备: %s, 温度: %d°C, 电压: %.2fV", device_name, temperature, voltage);
ESP_LOGI(TAG, "十六进制值: 0x%08X", 0x12345678);
ESP_LOGI(TAG, "二进制数据: %.*s", 8, "ABCDEFGH");
```

## 3. 日志级别配置

### 3.1 编译时配置

在 `menuconfig` 中配置：
```
Component config → Log output → Default log verbosity
```

### 3.2 运行时配置

```c
// 设置全局日志级别
esp_log_level_set("*", ESP_LOG_INFO);

// 设置特定模块的日志级别
esp_log_level_set("MY_MODULE", ESP_LOG_DEBUG);
esp_log_level_set("WIFI", ESP_LOG_WARN);
esp_log_level_set("BT", ESP_LOG_ERROR);
```

## 4. 高级功能

### 4.1 条件日志

```c
#if CONFIG_LOG_DEFAULT_LEVEL >= ESP_LOG_DEBUG
    ESP_LOGD(TAG, "这条消息只在调试级别或更高时编译");
#endif

// 或者使用内置宏
ESP_LOG_LEVEL_LOCAL(ESP_LOG_DEBUG, TAG, "本地调试消息");
```

### 4.2 自定义日志函数

```c
// 自定义日志输出函数
int custom_log_vprintf(const char *fmt, va_list args)
{
    // 添加时间戳
    printf("[%lu] ", esp_log_timestamp());
    return vprintf(fmt, args);
}

void app_main(void)
{
    // 设置自定义日志函数
    esp_log_set_vprintf(custom_log_vprintf);
}
```

### 4.3 颜色输出控制

```c
// 启用/禁用颜色输出
esp_log_set_color_enabled(true);   // 启用颜色
esp_log_set_color_enabled(false);  // 禁用颜色
```

## 5. 实用示例

### 5.1 错误处理与日志

```c
esp_err_t result = some_function();
if (result != ESP_OK) {
    ESP_LOGE(TAG, "函数执行失败: %s", esp_err_to_name(result));
    return result;
}
ESP_LOGI(TAG, "函数执行成功");
```

### 5.2 调试网络连接

```c
static const char *TAG = "WIFI";

void wifi_event_handler(void* arg, esp_event_base_t event_base,
                       int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        ESP_LOGI(TAG, "WiFi 站点模式启动");
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        ESP_LOGW(TAG, "WiFi 连接断开，尝试重连...");
        esp_wifi_connect();
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "获得IP地址:" IPSTR, IP2STR(&event->ip_info.ip));
    }
}
```

### 5.3 性能监控

```c
static const char *TAG = "PERF";

void performance_monitor(void)
{
    uint32_t free_heap = esp_get_free_heap_size();
    uint32_t min_free_heap = esp_get_minimum_free_heap_size();
    
    ESP_LOGI(TAG, "可用堆内存: %lu bytes", free_heap);
    ESP_LOGI(TAG, "最小可用堆内存: %lu bytes", min_free_heap);
    
    if (free_heap < 10000) {
        ESP_LOGW(TAG, "堆内存不足！");
    }
}
```

## 6. 最佳实践

### 6.1 合理使用日志级别

- **ESP_LOGE**: 严重错误，必须修复
- **ESP_LOGW**: 潜在问题，建议关注
- **ESP_LOGI**: 重要状态变化
- **ESP_LOGD**: 调试信息，发布时关闭
- **ESP_LOGV**: 详细调试，仅开发时使用

### 6.2 标签命名规范

```c
// 推荐的标签命名
static const char *TAG = "MAIN";      // 主程序
static const char *TAG = "WIFI_MGR";  // WiFi管理器
static const char *TAG = "SENSOR";    // 传感器模块
static const char *TAG = "BLE_ADV";   // BLE广播
```

### 6.3 生产环境配置

```c
void configure_production_logging(void)
{
    // 生产环境只保留错误和警告
    esp_log_level_set("*", ESP_LOG_WARN);
    
    // 关键模块保留信息级别
    esp_log_level_set("MAIN", ESP_LOG_INFO);
    esp_log_level_set("SYSTEM", ESP_LOG_INFO);
}
```

## 7. 常用配置选项

在 `sdkconfig` 中的相关配置：

```
CONFIG_LOG_DEFAULT_LEVEL=3          # 默认日志级别
CONFIG_LOG_DEFAULT_LEVEL_INFO=y     # 信息级别
CONFIG_LOG_MAXIMUM_EQUALS_DEFAULT=y # 最大级别等于默认级别
CONFIG_LOG_MAXIMUM_LEVEL=4          # 最大日志级别
CONFIG_LOG_COLORS=y                 # 启用颜色输出
CONFIG_LOG_TIMESTAMP_SOURCE_RTOS=y  # 使用RTOS时间戳
```

## 8. 发布版本去除日志

### 8.1 编译时完全去除日志

在发布版本中，可以通过配置完全去除日志代码，节省Flash空间和提高性能。

#### 方法一：通过 menuconfig 配置

```bash
idf.py menuconfig
```

导航到：
```
Component config → Log output → Default log verbosity → No output
```

或者设置为 Error only：
```
Component config → Log output → Default log verbosity → Error
```

#### 方法二：直接修改 sdkconfig

```
CONFIG_LOG_DEFAULT_LEVEL_NONE=y
CONFIG_LOG_DEFAULT_LEVEL=0
CONFIG_LOG_MAXIMUM_LEVEL=0
```

#### 方法三：通过 CMake 定义

在项目的 `CMakeLists.txt` 中添加：

```cmake
# 发布版本完全禁用日志
if(CONFIG_RELEASE_BUILD)
    target_compile_definitions(${COMPONENT_LIB} PRIVATE
        LOG_LOCAL_LEVEL=ESP_LOG_NONE
        CONFIG_LOG_DEFAULT_LEVEL=0
    )
endif()
```

### 8.2 条件编译去除特定日志

#### 使用预定义宏

```c
#include "esp_log.h"

static const char *TAG = "MY_MODULE";

void example_function(void)
{
    // 这些日志在发布版本中会被完全去除
    #if CONFIG_LOG_DEFAULT_LEVEL >= ESP_LOG_ERROR
        ESP_LOGE(TAG, "错误消息 - 只在错误级别及以上编译");
    #endif
    
    #if CONFIG_LOG_DEFAULT_LEVEL >= ESP_LOG_WARN
        ESP_LOGW(TAG, "警告消息 - 只在警告级别及以上编译");
    #endif
    
    #if CONFIG_LOG_DEFAULT_LEVEL >= ESP_LOG_INFO
        ESP_LOGI(TAG, "信息消息 - 只在信息级别及以上编译");
    #endif
    
    #if CONFIG_LOG_DEFAULT_LEVEL >= ESP_LOG_DEBUG
        ESP_LOGD(TAG, "调试消息 - 只在调试级别及以上编译");
    #endif
}
```

#### 自定义发布宏

```c
// 在项目头文件中定义
#ifdef CONFIG_RELEASE_BUILD
    #define RELEASE_LOG_LEVEL ESP_LOG_ERROR
#else
    #define RELEASE_LOG_LEVEL ESP_LOG_VERBOSE
#endif

// 使用自定义宏
#if RELEASE_LOG_LEVEL >= ESP_LOG_INFO
    ESP_LOGI(TAG, "这条消息在发布版本中会被去除");
#endif
```

### 8.3 创建发布配置文件

#### 创建 sdkconfig.release

```bash
# sdkconfig.release - 发布版本配置
CONFIG_LOG_DEFAULT_LEVEL_ERROR=y
CONFIG_LOG_DEFAULT_LEVEL=1
CONFIG_LOG_MAXIMUM_LEVEL=1
CONFIG_LOG_COLORS=n
CONFIG_FREERTOS_ASSERT_ON_UNTESTED_FUNCTION=n
CONFIG_OPTIMIZATION_LEVEL_RELEASE=y
CONFIG_COMPILER_OPTIMIZATION_SIZE=y
```

#### 使用发布配置构建

```bash
# 复制发布配置
cp sdkconfig.release sdkconfig

# 构建发布版本
idf.py build

# 或者直接指定配置文件
idf.py -D SDKCONFIG=sdkconfig.release build
```

### 8.4 自动化发布脚本

#### build_release.sh (Linux/macOS)

```bash
#!/bin/bash

echo "构建发布版本..."

# 备份当前配置
cp sdkconfig sdkconfig.backup

# 使用发布配置
cp sdkconfig.release sdkconfig

# 清理并构建
idf.py fullclean
idf.py build

# 创建发布目录
mkdir -p release
cp build/*.bin release/
cp build/*.elf release/

# 恢复原配置
cp sdkconfig.backup sdkconfig

echo "发布版本构建完成，文件位于 release/ 目录"
```

#### build_release.bat (Windows)

```batch
@echo off
echo 构建发布版本...

:: 备份当前配置
copy sdkconfig sdkconfig.backup

:: 使用发布配置
copy sdkconfig.release sdkconfig

:: 清理并构建
idf.py fullclean
idf.py build

:: 创建发布目录
if not exist release mkdir release
copy build\*.bin release\
copy build\*.elf release\

:: 恢复原配置
copy sdkconfig.backup sdkconfig

echo 发布版本构建完成，文件位于 release\ 目录
```

### 8.5 验证日志去除效果

#### 检查二进制大小

```bash
# 构建调试版本
idf.py build
ls -la build/*.bin

# 构建发布版本
cp sdkconfig.release sdkconfig
idf.py build
ls -la build/*.bin

# 比较大小差异
```

#### 检查符号表

```bash
# 检查是否还有日志相关符号
xtensa-esp32-elf-nm build/project_name.elf | grep -i log
xtensa-esp32-elf-objdump -t build/project_name.elf | grep -i log
```

#### 运行时验证

```c
void verify_log_removal(void)
{
    ESP_LOGI("TEST", "如果看到这条消息，说明日志未被完全去除");
    printf("正常的printf输出仍然可用\n");
}
```

### 8.6 保留关键日志

在某些情况下，您可能希望在发布版本中保留一些关键的错误日志：

```c
// 强制保留的关键日志
#define CRITICAL_LOG(tag, format, ...) \
    do { \
        printf("CRITICAL [%s]: " format "\n", tag, ##__VA_ARGS__); \
    } while(0)

// 使用示例
void critical_function(void)
{
    esp_err_t err = some_critical_operation();
    if (err != ESP_OK) {
        // 这个日志即使在发布版本中也会保留
        CRITICAL_LOG(TAG, "关键操作失败: %s", esp_err_to_name(err));
    }
}
```

## 9. 注意事项

1. **性能影响**: 日志输出会影响性能，特别是在高频调用的函数中
2. **内存使用**: 日志字符串会占用Flash空间
3. **串口速度**: 确保串口波特率足够高以避免日志输出阻塞
4. **线程安全**: ESP-LOG 是线程安全的，可以在多任务环境中使用
5. **发布版本**: 在生产环境中应该去除调试日志以节省空间和提高性能

## 10. 调试技巧

### 10.1 动态调整日志级别

```c
// 通过串口命令动态调整
void process_log_command(char *command)
{
    if (strncmp(command, "log_level ", 10) == 0) {
        char *tag = command + 10;
        char *level_str = strchr(tag, ' ');
        if (level_str) {
            *level_str = '\0';
            level_str++;
            int level = atoi(level_str);
            esp_log_level_set(tag, level);
            ESP_LOGI(TAG, "设置 %s 日志级别为 %d", tag, level);
        }
    }
}
```

### 10.2 内存泄漏检测

```c
static const char *TAG = "MEM_DEBUG";

void check_memory_leak(void)
{
    static uint32_t last_free_heap = 0;
    uint32_t current_free_heap = esp_get_free_heap_size();
    
    if (last_free_heap != 0) {
        int32_t diff = current_free_heap - last_free_heap;
        if (diff < 0) {
            ESP_LOGW(TAG, "内存减少: %ld bytes", -diff);
        }
    }
    last_free_heap = current_free_heap;
}
```

这个日志系统是ESP32开发中非常重要的调试工具，合理使用可以大大提高开发效率和程序的可维护性。