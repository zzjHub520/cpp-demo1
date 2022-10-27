### [使用CMake](https://www.thinbug.com/q/41345844)

时间：2016-12-27 13:11:12

标签： c++ cmake spdlog nlohmann-json



我有一个简单的项目，需要三个仅限标题的库才能进行编译：[websocketpp](https://github.com/zaphoyd/websocketpp)，[spdlog](https://github.com/gabime/spdlog)和[nlohmann/json](https://github.com/nlohmann/json)。

项目结构如下：

```
└── src
    ├── app
    │   ├── CMakeLists.txt
    │   ├── src
    │   └── test
    ├── CMakeLists.txt
    ├── core
    │   ├── CMakeLists.txt
    │   ├── include
    │   ├── src
    │   └── test
    └── vendor
        ├── install.cmake
        ├── nlohmann_json
        ├── spdlog
        └── websocketpp
```

根CMakeLists.txt如下：

```
cmake_minimum_required(VERSION 3.6.1 FATAL_ERROR)

..

# External 3rd party libs that we include

include(vendor/install.cmake)

add_subdirectory(core)
add_subdirectory(app)
```

这个想法基本上是每个子目录都是一个库（例如`core`），而`app`＆＃34;聚合＆＃34;他们都是。每个库（例如`core`）都是这样构建的（`core/CMakeLists.txt`）：

```
project(foo-core VERSION 0.1 LANGUAGES CXX)
add_library(foo-core
  src/foobar/foobar.cc
  src/foobaz/baz.cc)

target_include_directories(foo-core PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
  PRIVATE src)

target_link_libraries(foo-core websocketpp spdlog) # <- see here, using spdlog & websocketpp

# 'make install' to the correct location

install(TARGETS foo-core EXPORT FooCoreConfig
  ARCHIVE  DESTINATION lib
  LIBRARY  DESTINATION lib
  RUNTIME  DESTINATION bin)
install(DIRECTORY include/ DESTINATION include)

install(EXPORT FooCoreConfig DESTINATION share/FooCore/cmake)

export(TARGETS foo-core FILE FooCoreConfig.cmake)
```

注意我如何链接依赖项（它们是仅限标题的库！）。这是我获取它们的方式（`vendor/install.cmake`）：

```
# spdlog

if((NOT SPDLOG_INCLUDE_DIR) OR (NOT EXISTS ${SPDLOG_INCLUDE_DIR}))
  message("Unable to find spdlog, cloning...")

  execute_process(COMMAND git submodule update --init -- vendor/spdlog
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

  set(SPDLOG_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/vendor/spdlog/include/
    CACHE PATH "spdlog include directory")

  install(DIRECTORY ${SPDLOG_INCLUDE_DIR}/spdlog DESTINATION include)

  # Setup a target

  add_library(spdlog INTERFACE)
  target_include_directories(spdlog INTERFACE
    $<BUILD_INTERFACE:${SPDLOG_INCLUDE_DIR}>
    $<INSTALL_INTERFACE:include>)

  install(TARGETS spdlog EXPORT spdlog DESTINATION include)
endif()

# websocketpp

if((NOT WEBSOCKETPP_INCLUDE_DIR) OR (NOT EXISTS ${WEBSOCKETPP_INCLUDE_DIR}))
  message("Unable to find websocketpp, cloning...")

  execute_process(COMMAND git submodule update --init -- vendor/websocketpp
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

  set(WEBSOCKETPP_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/vendor/websocketpp/
    CACHE PATH "websocketpp include directory")

  install(DIRECTORY ${WEBSOCKETPP_INCLUDE_DIR}/websocketpp DESTINATION include)

  # Setup a target

  add_library(websocketpp INTERFACE)
  target_include_directories(websocketpp INTERFACE
    $<BUILD_INTERFACE:${WEBSOCKETPP_INCLUDE_DIR}>
    $<INSTALL_INTERFACE:include>)

  install(TARGETS websocketpp EXPORT websocketpp DESTINATION include)
endif()

# nlohmann/json

if((NOT NLOHMANN_JSON_INCLUDE_DIR) OR (NOT EXISTS ${NLOHMANN_JSON_INCLUDE_DIR}))    
  message("Unable to find nlohmann/json, cloning...")

  execute_process(COMMAND git submodule update --init -- vendor/nlohmann_json
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

  set(NLOHMANN_JSON_INCLUDE_DIR
    ${CMAKE_CURRENT_SOURCE_DIR}/vendor/nlohmann_json/src/
    CACHE PATH "nlohmann/json include directory")

  install(FILES ${NLOHMANN_JSON_INCLUDE_DIR}/json.hpp DESTINATION include)

  # Setup a target

  add_library(nlohmann_json INTERFACE )
  target_include_directories(nlohmann_json INTERFACE
    $<BUILD_INTERFACE:${NLOHMANN_JSON_INCLUDE_DIR}>
    $<INSTALL_INTERFACE:include>)

  install(TARGETS nlohmann_json EXPORT nlohmann_json DESTINATION include)
endif()
```

到目前为止一切顺利：您可以看到依赖项被作为git子模块获取，这值得管理它们。但是，当我用`mkdir build && cd build && cmake ../src`编译项目时，我有以下错误：

> CMake错误：安装（EXPORT FooCoreConfig ...）包含目标  foo-core，它需要不在导出中的目标websocketpp  集。
>
> CMake错误：安装（EXPORT FooCoreConfig ...）包含目标  foo-core，它需要不在导出集中的目标spdlog。

包括标题，例如`#include <spdlog/spdlog.h>`或`#include <nlohmann/json.hpp>`会产生错误，指出未找到标题。

说实话，我对CMake不太满意，过去两天我都在调试这个。这可能是非常简单的事情，但我无法想象如何实现它。实际上，人们只会传递-I作为编译器标志来使用我想要的库，但CMake抽象似乎让我感到困惑。如果有人可以解释为什么这不起作用，并且希望将这些库包含在我的项目中的正确方法，我将非常高兴。提前谢谢！

#### 1 个答案:

答案 0 :(得分：1)

就像你说的那样：你没有在导出集中安装你的目标。换句话说，您错过了仅限标题目标的`install(EXPORT ...`行。例如，考虑到仅标题库`websocketpp`，您应该：

```
add_library(websocketpp INTERFACE)
target_include_directories(websocketpp INTERFACE
  $<BUILD_INTERFACE:${WEBSOCKETPP_INCLUDE_DIR}>
  $<INSTALL_INTERFACE:include>)

install(TARGETS websocketpp EXPORT websocketpp-config DESTINATION include)

# here is the missing line:

install(EXPORT websocketpp-config DESTINATION share/websocketpp/cmake)
```

其他图书馆也一样。