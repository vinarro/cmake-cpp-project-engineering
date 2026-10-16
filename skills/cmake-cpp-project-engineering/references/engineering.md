# 工程基础

本文件覆盖单 Target 和多 Target 工程；公开接口、DLL 导出、测试和子项目规则仅在相应行为存在时应用。

## 核心决策

- Target 来自真实边界：独立职责、复用、依赖 ownership、测试、consumer 或 packaging/export。
- Target graph 说明谁依赖谁、哪些 requirement 传播、哪些属于实现及消费者契约。
- 源码、资源、生成步骤和依赖按实际 ownership 接线；目录与 CMakeLists 边界随 Target graph 演进。
- 按当前需求建立结构，不预设模板、目录树、Target 数量、依赖管理器或 CMakeLists 数量；使用适用的现代能力，不引入尚无需求的基础设施。

## Target 与接口

按 requirement 的实际使用方选择 PRIVATE/PUBLIC/INTERFACE，区分实现依赖与需传播的 consumer contract。

### DLL 与插件导出

Windows SHARED target 即使内部使用，也须实际导出跨 DLL 符号以生成可用 import library。
按接口选择显式导出宏、GenerateExportHeader 或适用的自动导出；PUBLIC 与 CMake package export 不能代替二进制符号导出。
MODULE 插件按 ABI 导出入口；是否交付 SDK 与内部 DLL 导出是独立决策。

### Source Declaration

非 trivial Target 优先分开 Target 定义和 source ownership：

```cmake
add_library(foo_core)
target_sources(foo_core PRIVATE parser.cpp engine.cpp)
```

简单 Target 允许 `add_executable(foo src/main.cpp)`，不为形式拆开。

默认显式列出所属源码，不用 file(GLOB/GLOB_RECURSE) 收集项目源文件。仅在确有动态枚举语义时考虑 GLOB。

### Public Headers

真实 public header contract 推荐用 FILE_SET HEADERS 表达头文件归属、include interface 和安装 ownership：

```cmake
target_sources(foo_core
    PUBLIC FILE_SET HEADERS
    BASE_DIRS "${PROJECT_SOURCE_DIR}/include"
    FILES "${PROJECT_SOURCE_DIR}/include/foo/core/parser.hpp"
)
```

示例要求 PROJECT_SOURCE_DIR 是该库自己的项目根；作为 subproject 时不把 CMAKE_SOURCE_DIR 当本库根。
相对路径按当前 CMakeLists 目录解释，避免局部文件中的 include 意外指向 src/core/include。
BASE_DIRS 对应实际 include 前缀；共享 include 根的多个组件仍分别列出自己拥有的头文件。
executable、internal-only library 或没有 public API 的 Target 不必为形式使用 file set。

### Component Naming

定义 project key 与 component key。目录用短组件名（src/core、src/image），physical library target 带项目前缀（foo_core、foo_image），避免 subproject 名称碰撞。
有意义的 library interface 提供 namespaced ALIAS，内部消费者也优先使用该 canonical interface：

```cmake
add_library(foo_core)
add_library(foo::core ALIAS foo_core)
target_link_libraries(foo_cli PRIVATE foo::core)
```

build-tree ALIAS 不被安装，安装后的 consumer 命名和 export 规则见 [SDK/package](delivery.md#sdk-与-package)。

只有一个主要库时可用 physical target foo 与 alias foo::foo，不机械产生 foo_foo。
源码可直接放 src、公开头放 include/foo；形成多个组件后再演化为 foo_core → foo::core 等。

executable 不要求 namespaced alias。logical target 与产物名可不同，如 target foo_cli 用 `OUTPUT_NAME foo` 生成 foo。

## 目录与 Ownership

### Canonical Directory Semantics

仅在有需求时创建目录：`include/` 放 consumer-facing public headers，`src/` 放产品实现，`tests/`、`benchmarks/`、`examples/` 放独立消费者，`cmake/` 放复用的 CMake 机制。根目录通常提供 CMakeLists 和共享 Presets；这不是强制目录树。

产品实现按 Target/component 分组，如 `src/core`、`src/image`、`src/cli`，避免默认用 utils/helpers/common/managers 等模糊类别。
公开头文件体现项目与组件，如 `include/foo/core/parser.hpp` 对应 `#include <foo/core/parser.hpp>`；不强制 C++ namespace 设计。
私有头文件靠近实现，如 `src/core/parser_impl.hpp`，不机械地将所有头文件放 include、所有 cpp 放 src。

### CMakeLists.txt Placement

CMakeLists 按模块职责和可读性组织。有明确构建或维护职责的模块，通常由局部 CMakeLists 管理其源码、编译要求和依赖；承担组合职责的上层 CMakeLists 负责最终库或公共聚合入口的组合。是否拆分 CMakeLists，不由最终交付库的数量决定。
目录存在不等于必须有 CMakeLists；同一 cohesive Target 的多个源码目录可由同一文件管理。

简单单 Target 工程可在根 CMakeLists 直接定义 Target，不为“根必须薄”制造子目录。
include 用于复用的 CMake 机制，不默认用 `cmake/FooCore.cmake` 等 Target 定义文件模拟组件。
测试入口的顶层／子项目规则见 [测试与子项目](#测试与子项目)。

### Other Build-Relevant Files

icons、images、shaders、Qt resources、Windows .rc、macOS plist、schemas、runtime data、plugin manifests 和 generated inputs 都先判断 ownership。
Target-local 内容靠近组件，例如 `src/gui/resources/app.ico`；多 Target 共享时可用语义明确的项目目录，如 `resources/shaders`；平台专用内容可放 `src/gui/platform/windows`。不预建所有资源类型目录。

### Files Outside CMake Responsibility

不影响 build/install/package/runtime wiring/Target ownership 的文件不主动重组。
包括 .clang-format、.clang-tidy、.vscode、文档、设计文件和笔记；保留用户／项目选择或交由对应 Skill。

### Generated Files

configured headers、codegen/protobuf 输出及 generated metadata 默认进入 build tree。
建立干净构建所需的生成依赖，不默认污染 `src/generated` 或 `include/generated`；有意纳入版本管理的生成源码可以例外。

### Recommended Layout Evolution

同一结构随 Target graph 演进，而非选择不同固定模板：

| 实际需求 | 对应结构 |
|---|---|
| 简单单 Target | 根 CMakeLists、CMakePresets、src/main.cpp 即可 |
| CLI + core | src/core、src/cli 各自管理 Target；有公开接口才建 include/foo/core，有测试才建 tests |
| 多组件 | 按职责增加 src/image、src/network 及真实公开头文件；测试／benchmark 也按独立消费者边界展开 |

## 语言与配置

### Minimum CMake Version

没有特殊兼容要求的新工程默认 `cmake_minimum_required(VERSION 3.23)`，该 baseline 支持 Presets、PROJECT_IS_TOP_LEVEL 与 FILE_SET HEADERS。
不默认声明未经验证的 `3.23...<max>` policy maximum。
更低版本要求由真实环境兼容性与所需功能共同决定；REDESIGN 不天然保留旧 minimum。

### `project()`

纯 C++ 工程用 `project(foo LANGUAGES CXX)`，避免无意启用 C。
library、package 或有版本语义的应用可添加 VERSION；不为模板完整性给所有工程强制加版本。

### C++ Language Requirements

优先用 target_compile_features 表达语言要求并按接口传播，不默认用全局 CMAKE_CXX_STANDARD 掩盖各 Target 的真实要求。
实际标准由项目／用户要求和编译器能力决定，不能从 CMake 版本推断编译器支持。

### Build Configuration Scope

使用准确表达 requirement ownership 的最窄正确 scope，通常是 target_compile_features/options/definitions、target_include_directories/link_libraries/link_options。
若要求确实属于整个目录子树、toolchain、platform 或 project-wide configuration，更高层 scope 合理；不把 target scope 当绝对规则。

## Presets

新工程通常以 CMakePresets.json 作为共享构建入口。
CMakeUserPresets.json 放开发者路径、本机 SDK、个人 generator 和 toolchain，默认不提交。

### Presets 应保持最小

仅创建有真实开发价值的 configure/build/test 入口，不预设大量配置、编译器和 sanitizer 组合。
声明 CMake 3.23 时，Presets 使用其支持的 schema 1–4 及字段；不为样例强制选最高 schema。
workflow presets、新 API 或包管理集成需另核 baseline，不能让 Presets 实际要求高于宣称的 minimum。

### Generator-aware Presets

Ninja、Unix Makefiles 等 single-config generator 的 Debug/Release 通常在 configure 阶段选择。
Visual Studio、Xcode、Ninja Multi-Config 通常使用一个 configure preset 加多个 build/test configuration。

Multi-config 的 build/test preset 必须选择一致的 configuration；CTest 不会因 build 默认选了 Debug 自动推断。
省略 generator 不代表 single-config；支持 Windows 默认 Visual Studio 时也需正确的多配置入口。

## 测试与子项目

仅在实际存在测试、benchmark、example 或子项目复用需求时应用本节，不预建这些目录。

### Tests Layout

测试是独立 correctness consumer，通常放与 src 同级的 tests。
单 test Target 用 tests/CMakeLists.txt 足够；多个独立 test Target 才按 core/image/integration 等 ownership 分层。
不默认放 src/core/tests，除非大型模块化工程确有 component-local test ownership。

### Benchmarks 与 Examples

存在性能消费者或示例／文档消费者时，分别使用与 tests 同级的 benchmarks 和 examples。不存在需求不创建，默认也不安装；显式交付规则见 [delivery.md](delivery.md)。

### Subproject Friendliness

可复用工程须兼顾顶层配置及 add_subdirectory/FetchContent 消费。
作为子项目默认不开自身 tests、benchmarks、examples，不覆盖父工程 flags、语言标准、prefix 或其他选项。

**CTest 根入口由顶层工程负责。** 可复用子项目不得无条件 include(CTest)，也不因父工程 BUILD_TESTING=ON 自动开启自身测试。
使用 project-scoped 测试选项：顶层默认跟随 BUILD_TESTING，子项目默认 OFF，显式设置该项目选项优先。
顶层可 include(CTest)；显式开启自身测试时确保顶层已 enable_testing。作为子项目时由 parent 建立根测试入口，子项目只注册自己获准的 tests，不强行开启父工程测试。

```cmake
if(PROJECT_IS_TOP_LEVEL)
    include(CTest)
    set(foo_tests_default "${BUILD_TESTING}")
else()
    set(foo_tests_default OFF)
endif()
option(FOO_BUILD_TESTING "Build foo tests" "${foo_tests_default}")
if(FOO_BUILD_TESTING)
    if(PROJECT_IS_TOP_LEVEL)
        enable_testing()
    endif()
    add_subdirectory(tests)
endif()
```

示例放在本项目 project() 之后，option 名按项目命名。

## 验证

| 受影响的行为 | 最低工程证据 |
|---|---|
| CMake 语法、配置 | 用相关配置完成 configure/generate；这不能证明已编译或链接 |
| 源码声明、编译选项、生成文件 | 构建受影响 Target；codegen 需要干净 build tree |
| 链接关系或 PUBLIC usage requirements | 构建实际或最小代表 consumer，验证编译和链接；判断接口传播时，以该 consumer 实际获得的编译、链接要求为依据。PRIVATE 不等于永远不会影响静态库的最终链接 |
| 公开头文件 | consumer 仅依靠 canonical target 编译，避免 global includes 掩盖缺失的接口要求 |
| Presets | 实际调用相关 configure/build/test preset；仅 JSON 解析、--list-presets 或 configure 成功不足以验收完整入口 |
| CTest | 构建测试 Target、枚举注册项、执行相关测试；枚举成功不等于测试通过 |
| 子项目行为 | 由干净 parent 消费公开 Target，验证自身测试默认关闭／显式开启，parent prefix、语言标准、flags 和其他选项不被改变；顶层单独成功不足以证明可嵌入 |

动态依赖测试通过工程声明的 runtime staging 或对应 Target/configuration 的测试环境运行，不依赖开发者碰巧配置的 SDK PATH；测试环境不能替代正式部署。
为环境建立本地配置时不向共享 Presets 写入机器绝对路径。
