# 运行时部署

先按 [交付](delivery.md) 确定 Delivery Surface、prefix 和各应用目录；运行时部署沿用这些目录决策。

## Build Tree 与交付目录

`CMAKE_RUNTIME_OUTPUT_DIRECTORY` 等描述 build tree 产物位置，`install()` 描述 install tree。
build/bin 能运行不等于交付物可以独立运行。

项目自有 shared library 若被应用运行时需要，也应显式安装其 runtime artifact。
`RUNTIME_DEPENDENCIES` 不会把 build tree 中另一个项目自有 Target 自动当作第三方运行时依赖安装。

## 部署方式

目标环境不能假定已提供全部运行时依赖时，建立部署方案：

| 顺序 | 采用条件 |
|---|---|
| Dependency/framework 专用机制 | 优先遵循适用 recipe 或 framework 的部署机制，处理动态库之外的 plugins、平台集成文件、translations、runtime metadata、compiler runtime |
| 通用 CMake install-time runtime dependency handling | 没有更合适的专用部署机制时使用 |
| 定制 fallback | Target metadata 或通用 runtime scanning 不足时才使用 |

选定的 helper 或 vendor tool 应接入 [入口认可的正式交付流程](../SKILL.md#验证与交付)。
已有 framework CMake helper 时应按适用 recipe 接入，不绕过它后靠会话中的临时复制修补。
通用规则不硬编码 Qt、OpenCV、PCL 等具体库的命令；这些知识保留在各自 recipe 中，不能靠通用 DLL 扫描猜测 framework 的全部运行需求。

## Helper 与安装脚本

使用 helper 前核对其 executable、runtime libraries、plugins/resources 路径与实际应用目录一致。
生成安装脚本时区分 configure、generate、install 阶段：已知相对应用路径优先在配置时确定，generator expressions 在 generate 阶段求值，install-time prefix 保留到安装阶段。

需要延迟求值的变量须核对 helper 是否再次展开字符串。
第三方 helper 通过 macro 转发 CONTENT 时可能再次展开，单层转义不证明保留成功；核对 SDK 实际接口，使用其支持的直接函数或模板生成方式。
检查生成脚本中的实际 EXE 路径，防止空变量形成 `/app.exe` 或部署到默认 bin；不能仅凭源码中写了转义认定结果正确。
同时检查 configure-time prefix 未提前固化，安装时更换 prefix 后路径仍正确。

高层 helper 的默认布局不能满足明确要求时，在正式流程中使用 framework 支持的定制接口并说明调整；
不能为了迁就默认值把 EXE 改回 bin，也不能靠临时手工复制弥补目录错位。

## 通用运行时依赖处理

没有更适合的 framework-specific 机制时，可以使用 CMake 3.21+ 的 install-time 机制：

```cmake
install(TARGETS <target>
    RUNTIME_DEPENDENCIES
        ...
    RUNTIME DESTINATION "${app_install_dir}"
    LIBRARY DESTINATION "${app_runtime_library_dir}"
)
```

也可以显式使用 `RUNTIME_DEPENDENCY_SET`。示例中的目录按应用布局和平台加载规则确定，不能无条件复用 SDK 的 bin/lib。
按目标平台和依赖知识配置 DIRECTORIES、PRE_INCLUDE/PRE_EXCLUDE、POST_INCLUDE/POST_EXCLUDE regexes。
Windows SDK/vendor DLL 搜索目录优先来自适用 recipe 或明确 SDK 元数据，不在通用规则中维护机器路径。

## Build-tree DLL 便利机制

`$<TARGET_RUNTIME_DLLS:tgt>` 是 generator expression，不是 `install(TARGET_RUNTIME_DLLS)` 子命令。
它适合在 DLL 平台构建后将 Target graph 已描述的 DLL 复制到 executable 旁，方便本地运行。
它依赖 imported/shared Targets 正确记录 DLL location；第三方 package metadata 不完整时不能假定它能找到所有运行时文件。
交付运行闭包仍按上面的部署机制处理，除非项目或 recipe 明确选择其他方式。

## Fallback

必要时可使用 `file(GET_RUNTIME_DEPENDENCIES)` 配合 install(CODE/SCRIPT)、install(DIRECTORY)，或在正式交付流程中调用 framework/vendor deploy tool。
Fallback 必须处理 configuration、architecture、Debug/Release、版本和 stale files 的匹配。
这些特殊知识优先放入对应 recipe，不成为所有项目的默认样板。

## 验证

- 按正式流程从干净输出目录重建 runtime closure/framework 部署结果；实际运行环境不依赖 provider build tree 或开发者的 SDK PATH。
- CMake 安装流程需实际使用不同的 `cmake --install --prefix` 安装并启动，核对 EXE、DLL、plugins 和资源均跟随当前目录决策；外部打包流程同样检查其实际交付目录。
- Smoke 必须触发待验证的加载路径。GUI/QML/plugin 部署不能用 framework 初始化前就返回的纯计算检查替代；隐藏窗口也应初始化真实平台插件。
- 不强制使用交付物未包含的额外 headless 插件，再把检查环境的问题误报为部署失败。
