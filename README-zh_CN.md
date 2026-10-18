# STM32HAL CMake C++ 模板

中文 | [English](README.md)

此目录用于接入 STM32CubeMX 生成的 CMake 工程。`applications` 可单独复制；需要板级延时实现时再复制 `bsp`。

## 目录结构与头文件引用

用户代码按功能模块组织，每个模块的 `.cpp`、`.hpp`（或 `.c`、`.h`）放在同一个目录中：

```text
applications/
└── main/
    ├── cpp_interface.cpp
    └── cpp_interface.h
bsp/
└── boards/
    ├── common/
    │   └── struct_typedef.h
    └── delay/
        ├── bsp_delay.cpp
        └── bsp_delay.hpp
cmake/
└── user/
    └── CMakeLists.txt
```

`cmake/user/CMakeLists.txt` 将 `applications` 和 `bsp/boards` 加入目标的头文件搜索路径。引用头文件时，使用相对于这两个目录的“模块目录/头文件名”路径：

```cpp
#include "main/cpp_interface.h"
#include "delay/bsp_delay.hpp"
#include "common/struct_typedef.h"
```

在这两个目录下新增模块时，沿用上述引用方式即可，无需为每个模块单独添加头文件搜索路径。CubeMX 生成的 `Core`、`Drivers` 等目录继续保留原有结构。

## 接入方法

1. 在 CubeMX 生成的根目录 `CMakeLists.txt` 中，保留原有的 C 标准设置，并在 `project()` 前增加 C++ 标准设置：

```cmake
# Setup compiler settings
set(CMAKE_C_STANDARD 11 CACHE STRING "C language standard")
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)

set(CMAKE_CXX_STANDARD 20 CACHE STRING "C++ language standard")
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)
```

`CACHE STRING` 把标准值保存为可覆盖的构建配置；已有构建目录中的缓存值不会因这里的默认值改变而自动更新。`*_EXTENSIONS ON` 允许使用编译器扩展，GCC 通常会选择 `gnu11` / `gnu++20` 方言；设为 `OFF` 则请求 `c11` / `c++20` 方言。CubeMX 生成的 C 代码建议保留 `C_EXTENSIONS ON`；自行编写的 C++ 若不使用 GNU 专有语法，可以将 `CXX_EXTENSIONS` 设为 `OFF`。

2. 复制 `cmake/user`、`applications` 和 `bsp`，**必须在顶层 `CMakeLists.txt` 中加入 `cmake/user`**，建议放在 CubeMX 子目录之后：

```cmake
# Add STM32CubeMX generated sources
add_subdirectory(cmake/stm32cubemx)

# Add USER generated sources
add_subdirectory(cmake/user)
```

当前生成的 `project(${CMAKE_PROJECT_NAME})` 没有限定语言，CMake 默认启用 C 和 C++；`enable_language(C ASM)` 不会关闭 C++，所以这一行**不用改**。如果以后生成的 `project()` 显式写了 `LANGUAGES C ASM`，则需把 `CXX` 加进去。

若只接入应用入口，可以不复制 `bsp`，同时从 `cmake/user/CMakeLists.txt` 中移除 `Bsp_Boards_Src` 的定义及其在 `target_sources()` 中的引用，并移除 `bsp/boards` 头文件搜索路径。

3. 每新增一个 `.c` 或 `.cpp` 文件，手动把路径写进 `cmake/user/CMakeLists.txt` 中对应的 `Applications_Src` 或 `Bsp_Boards_Src` 列表。例如新增 `bsp/boards/led/bsp_led.cpp` 时，在 `Bsp_Boards_Src` 中添加 `${CMAKE_CURRENT_SOURCE_DIR}/../../bsp/boards/led/bsp_led.cpp`。`.h`、`.hpp` 不需要列为编译源文件。

4. 在 CubeMX 的 `main.c` 等 C 文件的 `USER CODE` 头文件区包含应用入口：

```c
#include "main/cpp_interface.h"
```

在合适的 `USER CODE` 区调用 `cpp_main()`。`applications/main/cpp_interface.h` 通过 `extern "C"` 声明供 C 文件调用的接口。`isRTOS` 在该头文件中默认设为 `0`，此时 `cpp_main()` 进入无限循环；使用 FreeRTOS 时须将其设为 `1` 并选择合适的调用位置。

5. `bsp_delay` 使用 Cortex-M3/M4 的 DWT 周期计数器。`Init(sysclk)` 的参数单位是 MHz，应在系统时钟配置完成后调用。延时是忙等待，不会改写 SysTick 或 HAL 的 `HAL_Delay`。

## 清理脚本

`clean.bash` 始终针对脚本所在目录，按白名单清理文件。直接运行只预览；确认清单后运行 `./clean.bash --apply` 才会删除。它会删除白名单之外的顶层文件，包括 CubeMX 生成的 `Core`、`Drivers`、`CMakeLists.txt`、构建目录及 IDE 设置；保留 `applications`、`bsp`、`cmake/user` 和根目录的 `.ioc` 文件。

作者：Tung Chia-hui · [tungchiahui.cn](https://www.tungchiahui.cn) · 预发布版本
