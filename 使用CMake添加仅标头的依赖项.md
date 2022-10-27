# 使用CMake添加仅标头的依赖项*[英] Adding header-only dependencies with CMake*

查看：131 发布时间：2021/4/19 20:52:30 [c++](https://www.it1352.com/tag/c%2b%2b) [cmake](https://www.it1352.com/tag/cmake) [spdlog](https://www.it1352.com/tag/spdlog) [nlohmann-json](https://www.it1352.com/tag/nlohmann-json)

[![万维广告联盟](ImageMarkDown/使用CMake添加仅标头的依赖项/6GjiSvATaQ7SgC0g52LqdxoqZrfIIr4rc5cDhmx6.jpg)](https://wwads.cn/click/bundle?code=7rWAAkacjBXcsa3rjwP2M2pGjLTenL&version=2.3)

[
**天翼云新客特惠🎁**
S3云主机1C2G低至**9.6元/月**.领💰**35000元**大礼包,先领再买更划算.](https://wwads.cn/click/bundle?code=7rWAAkacjBXcsa3rjwP2M2pGjLTenL&version=2.3)[![img]()广告](https://wwads.cn/?utm_source=property-197&utm_medium=footer)



本文介绍了使用CMake添加仅标头的依赖项的处理方法，对大家解决问题具有一定的参考价值，需要的朋友们下面随着小编来一起学习吧！

### 问题描述

我有一个简单的项目，该项目需要三个仅标头的库才能进行编译:[ websocketpp ](https://github.com/zaphoyd/websocketpp)，[ spdlog ](https://github.com/gabime/spdlog)和[ nlohmann/json ](https://github.com/nlohmann/json).

项目结构如下:

```stylus
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

CMakeLists.txt根目录如下:

```scss
cmake_minimum_required(VERSION 3.6.1 FATAL_ERROR)

..

# External 3rd party libs that we include

include(vendor/install.cmake)

add_subdirectory(core)
add_subdirectory(app)
```

该想法基本上是，每个子目录都是一个库(例如` core `)，并且` app `聚集"所有它们.每个库(例如` core `)的构建方式如下(` core/CMakeLists.txt `):

```cmake
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

注意如何链接依赖项(它们是仅标头的库！).这就是我获取它们的方式(` vendor/install.cmake `):

```cmake
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

到目前为止，效果很好:您可以看到依赖项已作为git子模块获取，幸运的是，这使得管理它们更加容易.但是，当我使用` mkdir build&&编译我的项目时光盘制作cmake ../src `，出现以下错误:

> CMake错误:安装(导出FooCoreConfig ...)包括目标foo-core，它需要不在导出中的目标websocketpp设置.

CMake错误:安装(导出FooCoreConfig ...)包括目标foo-core，它需要不在导出集中的目标spdlog.

包括标题，例如` #include< spdlog/spdlog.h> `或` #include< nlohmann/json.hpp> `会产生错误，指出找不到标题.

说实话，我对CMake不太满意，并且我花了整整两天的时间进行调试.这可能确实很简单，但是我不知道如何实现.实际上，可能只是将-I作为编译器标志传递来使用所需的库，但是CMake抽象似乎使我感到困惑.如果有人能解释为什么这行不通，我将非常高兴，并且希望将这些库包含到我的项目中的正确方法是什么.预先感谢！

### 推荐答案

就像您所说的:您没有在导出集中安装目标.换句话说，您缺少仅标头目标的` install(EXPORT ... `行.例如，考虑仅标头的库` websocketpp `有:

```cmake
add_library(websocketpp INTERFACE)
target_include_directories(websocketpp INTERFACE
  $<BUILD_INTERFACE:${WEBSOCKETPP_INCLUDE_DIR}>
  $<INSTALL_INTERFACE:include>)

install(TARGETS websocketpp EXPORT websocketpp-config DESTINATION include)

# here is the missing line:

install(EXPORT websocketpp-config DESTINATION share/websocketpp/cmake)
```

其他库也一样.

这篇关于使用CMake添加仅标头的依赖项的文章就介绍到这了，希望我们推荐的答案对大家有所帮助，也希望大家多多支持IT屋！