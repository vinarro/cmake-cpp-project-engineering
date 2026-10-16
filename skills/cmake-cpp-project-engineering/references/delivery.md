# 交付、安装布局与 SDK

## 交付面

默认安装交付的触发范围见 [入口阅读路径](../SKILL.md#阅读路径)：CREATE 新建产品应用，或 REDESIGN 的授权范围包含产品应用交付。
整个产品工程的重新设计包含该范围，局部组件重设计不自动包含；用户明确选择其他交付方式时遵循该方式。
在写 `install()` 前确定 Delivery Surface：项目真正对外交付哪些 Target 和 artifacts。
安装、运行时部署与 package/consumer interface 分别考虑；依赖获取、发现和消费仍由 [依赖](dependencies.md) 管理，具体库的特殊部署要求由其 recipe 表达。

| Target／产物 | 默认交付行为 |
|---|---|
| 产品 executable/application | 在上述适用范围内默认建立正式安装规则，交付 executable 及运行所需 artifacts/resources |
| 内部 static/object/interface library | 不安装 archive、headers 或 package interface |
| 交付面内产物所需的内部 shared library | 交付运行时 artifact，不因此交付开发 headers、import archive 或 package |
| 对外分发的 library/SDK | 交付实际 library artifacts 和 public interface，需要时提供 package/export |
| tests/benchmarks/examples | 默认不安装，除非用户明确要求 |

Build target graph 不等于交付面：library 的存在、PUBLIC/INTERFACE usage requirements、namespaced ALIAS、FILE_SET 或被内部 Target 消费，都不能单独推出它应安装。
纯应用即使拆成多个内部库，仍默认只交付应用及运行闭包，不自动变成含内部 archive、头文件和 package config 的开发包。
这里的内部实现不属于公开 consumer contract；公开头使消费者需要的依赖类型、头文件或使用要求，应按公开接口处理，不能仅因目录名为 internal 就隐藏必要要求。
项目自有的内部静态实现默认封装进对外交付的库，不让 SDK 消费者额外取得内部 archive；交付边界见 [SDK 公共组件与内部封装](#sdk-公共组件与内部封装)。显式用户要求及需保留的既有兼容契约仍按入口优先级处理。

下面的目录和 prefix 是用户选定的个人默认策略，按 [入口优先级](../SKILL.md#适用优先级) 应用。
CHANGE 只在任务影响交付时调整相关逻辑，不顺手改写安装体系或现有有意的布局。

## 默认 Install Prefix

新建或按推荐策略重新设计的顶层工程，用户未显式设置 prefix 时，默认使用项目内 `install`。
在 `project()` 之后设置：

```cmake
if(PROJECT_IS_TOP_LEVEL
   AND CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX
        "${PROJECT_SOURCE_DIR}/install"
        CACHE PATH "Install prefix" FORCE
    )
endif()
```

这个默认值也适用于 static、shared 和 header-only SDK。共享 preset 可以显式设置 prefix，但不能替代直接 `cmake -S ... -B ...` 时应生效的默认行为。
不覆盖用户显式 `CMAKE_INSTALL_PREFIX`，作为 subproject 时不修改 parent prefix。
用户可通过 configure-time prefix 或 `cmake --install ... --prefix ...` 选择其他共同安装根。
destination 默认保持 prefix-relative，不把 `${CMAKE_INSTALL_PREFIX}` 拼入每条规则，也不用 `../` 安装到 prefix 外；更换 prefix 应保持相同相对布局。

## 应用与 SDK 目录

CREATE/REDESIGN 在本次交付范围内默认按下表布局；CHANGE 保留本任务要求维持的已有布局。显式布局和 install-dir 覆盖优先。
所有 executable，包括 GUI、CLI、demo、example，默认放入各自应用或示例目录；只有用户明确选择 bin 布局时使用 bin。
纯库 DLL 可以位于 SDK runtime 目录，不意味着 executable 也应放入 bin。

以下路径均相对 install prefix：

| 交付类型 | 默认布局 |
|---|---|
| 一个产品应用，无对外 SDK | `<Project>/` 直接放 EXE 和运行时内容；产品名与项目名不符时使用实际产品名，不添加无必要的 GUI/CLI 后缀 |
| 多个独立产品应用，无对外 SDK | 各自使用有意义的产品名称目录，运行时闭包完整，不机械添加 GUI/CLI 后缀 |
| SDK 加产品应用 | SDK 占用 `<Project>/`，内部使用 include/lib 等目录；应用位于并列的 `<Project>_GUI`、`<Project>_CLI` 或其他有意义的目录，不嵌入 SDK |
| 明确要求安装的 demo/example | `examples/<example-name>/`，包含该示例需要的运行时文件 |
| 纯 library/SDK | `<Project>/` 内使用 GNUInstallDirs 的 include/lib 及必要 runtime/package 目录，不产生无关应用目录 |

表格只决定已进入交付面的产物布局，不改变测试、benchmark 和 example 的默认不安装行为。
保留平台要求的 bundle 内部结构，例如 macOS `.app`，不为应用目录规则将其展平。
混合交付先确定每个 SDK、应用和 example 的目录，再写各 Target 规则；不能把 SDK include/lib 放到共同 prefix 根下而把项目名目录给 GUI。
目录名须在目标文件系统上实际不同，不能仅以大小写区分，如 Windows 上的 `Foo/` 和 `foo/`。

单应用可以直接表达为：

```cmake
set(foo_app_install_dir "${PROJECT_NAME}")
install(TARGETS foo_cli
    RUNTIME DESTINATION "${foo_app_install_dir}"
)
```

每个应用的 EXE、DLL、plugins、translations、qml、资源和配置使用同一组目录决策，并满足平台加载规则。
不能只把 EXE 移出 bin 而让部署 helper 使用旧目录，也不能让应用依赖另一交付物目录中的 DLL。
涉及 shared runtime、framework helper 或安装脚本时读取 [运行时](runtime.md)。

## Target 安装与 GNUInstallDirs

属于交付面的项目自有 binary Target 使用 `install(TARGETS ...)`，按 artifact 类型选择目的地。
不用 `install(FILES "$<TARGET_FILE:...>")` 替代，以免绕过 RPATH、configuration 和 namelink 等处理。
内部 shared Target 可以只安装 RUNTIME/LIBRARY，不因其存在而安装 ARCHIVE、headers 或 package。

交付 SDK 时使用 `include(GNUInstallDirs)`，在 SDK 根下组合相对 destination，例如 `${PROJECT_NAME}/${CMAKE_INSTALL_INCLUDEDIR}`。
用户明确选择系统布局时遵循其要求，不强加项目名外层目录。
Framework helper 需要这些变量时也可使用，但 `CMAKE_INSTALL_BINDIR` 是可配置路径变量，不等于字面 bin。
纯应用应在 framework 捕获目录前设置实际应用目录并保留显式覆盖；混合应用/SDK 使用各自 destination 或受控局部 scope，避免全局污染 SDK 和 parent。

## SDK 与 Package

本节只适用于对外分发的 library/SDK。按其真实 artifact 类型安装，例如：

```cmake
set(foo_sdk_install_dir "${PROJECT_NAME}")
install(TARGETS foo_core
    RUNTIME DESTINATION "${foo_sdk_install_dir}/${CMAKE_INSTALL_BINDIR}"
    LIBRARY DESTINATION "${foo_sdk_install_dir}/${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${foo_sdk_install_dir}/${CMAKE_INSTALL_LIBDIR}"
)
```

公开头属于 installed consumer contract 时，再安装对应 `FILE_SET HEADERS`。
SDK 的 FILE_SET、INSTALL_INTERFACE、export/config destination 使用一致的根目录，不能因内部库有 PUBLIC 接口就自动导出为 SDK。
安装 physical target，通过 `EXPORT_NAME` 与 export `NAMESPACE` 保持 build-tree 和 installed consumer 名一致：
例如 physical `foo_core`、EXPORT_NAME `core`、NAMESPACE `foo::`，两处都消费 `foo::core`。

Development package 表达导出 Target 的 usage requirements 和必要的 `find_dependency()`。
不因库可安装就默认打包所有第三方运行时；只有用户明确要求 self-contained SDK/runtime distribution 时，运行时打包才属于 package contract。

### SDK 公共组件与内部封装

SDK 可由一个或多个公开组件组成，并可提供聚合消费入口。公开组件按用户需求和已有公共接口确定；因内部解耦、复用或测试拆出的实现 Target，不因此成为公开组件或交付单元。

| 交付需求 | 应满足的结果 |
|---|---|
| 单个公共库 | 将约定范围内的项目自有内部实现实际封装进该库，消费者不需额外取得这些实现的内部 archive |
| 多个公共组件库 | 各组件提供自身完整消费要求，按契约独立使用；保留公共组件边界，单独使用时不引入无关组件 |
| 公共聚合入口 | 组合选定的公共组件并传递完整消费要求；保留组件库和约定的独立消费能力，不要求合并为一个二进制库 |

多组件 SDK 可以同时提供独立入口和聚合入口。仅实现当前需要的交付形态，不为未请求的形态新增切换选项、重复库产物或额外公共入口。
项目自有内部静态实现默认封装进对应公共库，不因提供聚合入口而交付内部 archive、私有头文件或开发接口；单个公共 Target 不能代替约定的单库二进制封装。
具体组合和封装方式按实际编译、链接及维护需求选择；保留用户要求的动态库、插件及既有公共组件边界，不擅自合并或重打包第三方依赖。

### Shared SDK 的内部依赖与导出

按目标平台和已有公共契约，分别满足公开接口、安装后链接和运行时依赖要求。内部动态依赖所需的产物与依赖信息须按交付范围完整提供，不因此公开私有开发接口。
保留用户要求的动态边界，并通过安装后独立消费者验证受影响配置；消费者不依赖生产者工作区、生产者导出宏或未交付的私有开发文件。
Windows 上，公开 DLL 依赖仅作运行时交付的内部 DLL，且消费者只需公开 DLL 的 import library 时，读取 [Windows 私有 DLL 的链接与导出](windows-shared-sdk.md)。

## 验证

- 先确定本任务实际生效的用户要求、已有项目策略或默认布局，按该布局验收；不把新工程个人默认值强加给 CHANGE。
- 默认 prefix 适用时，检查直接 configure 的默认值及显式覆盖。使用 CMake 安装时，从 fresh prefix 安装，并以不同 `--prefix` 核对相对布局；其他获认可的流程按 [入口的交付验证要求](../SKILL.md#验证与交付) 执行。
- 核对产品应用、SDK、example 的目录和交付面，确认未夹带未请求的内部开发产物；可行时从交付目录启动。build tree 能运行不能证明可部署。
- SDK 安装后 relocate，使原 provider source/build 不可访问，由只查找移动后 package 的独立 consumer 配置、构建、运行；仅安装成功或移动安装树但仍可访问原 provider，不足以证明可消费和无路径泄漏。
- 改动内部封装或 SDK 导出时，覆盖受影响的配置（多配置工程的 Debug/Release）：按约定验证单库封装、公共组件的独立消费及聚合消费；存在内部动态依赖时，验证其实际加载。核对内部开发产物未进入安装面，consumer 无需手工补接内部实现库、生产者宏或额外私有 include。
- 涉及运行闭包或 framework helper 时完成 [运行时验证](runtime.md#验证)，不能只检查 `install(TARGETS)`。
