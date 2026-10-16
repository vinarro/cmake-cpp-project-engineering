# Windows 私有 DLL 的链接与导出

仅在 Windows 上，公开 SDK DLL 依赖内部 DLL、消费者只需公开 DLL 的 import library，且内部 DLL 仅作运行时交付时读取。内部 DLL 的开发接口属于公共契约时，不适用本节的隐藏方式。
交付结果仍须满足 [Shared SDK 约束](delivery.md#shared-sdk-的内部依赖与导出)；以下是可选处理，不限定唯一接线方式。

`PRIVATE` 或 `$<BUILD_INTERFACE:...>` 不能单独保证内部 shared target 不进入导出依赖。检查生成的依赖属性及目标平台要求，不反复更换可见性来猜测结果。

满足上述条件时，可用配置相关的 import library 文件链接替代该内部 target 的直接链接：

```cmake
if(WIN32)
    target_link_libraries(foo PRIVATE "$<TARGET_LINKER_FILE:foo_engine>")
    add_dependencies(foo foo_engine)
endif()
```

该示例要求 `foo`、`foo_engine` 已定义为 SHARED，且各自具备正确的 DLL 导出声明。表达式按配置选择 import library，显式依赖保留构建顺序；不要并列保留已被替代的内部 target 链接。
文件链接不传播 `foo_engine` 的 usage requirements，须保留调用方真正需要的私有编译要求，不能复制生产者专用导出宏。
公开 DLL 按 SDK 安装和导出；内部 DLL 用 `install(TARGETS ... RUNTIME ...)` 正式交付，不附带内部 import library 或私有头文件。

也可采用满足相同交付约束的其他方案。用于描述必要动态依赖的辅助 imported target，不因出现在导出元数据中就成为公共开发接口；仍需核对实际传播的要求和安装产物。
不将上述文件链接方式直接套用于其他平台，也不为隐藏内部结构删除必要依赖信息或加载路径；按 [交付验证](delivery.md#验证) 核对安装后独立消费。
