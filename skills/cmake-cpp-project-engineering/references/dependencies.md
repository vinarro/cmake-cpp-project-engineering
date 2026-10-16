# 依赖获取、发现与消费

## 职责与放置位置

每个依赖分别考虑三层职责：

| 职责 | 解决的问题 | 放置位置 |
|---|---|---|
| 获取（provisioning） | 通过 submodule、FetchContent、包管理器、系统包或用户 SDK 满足依赖 | 能统一协调来源、版本和覆盖关系的 dependency authority scope |
| 发现（discovery） | 通过 find_package、add_subdirectory、pkg-config 或定制逻辑获得可消费 Target | 所有 direct consumers 都可见的最窄 CMake scope |
| 消费（consumption） | 具体 Target 的链接及使用要求 | 跟随 consumer Target，不为集中管理搬到 root |

例如只有 image 使用某库时在 image 范围发现；多个 sibling components 使用时放到共同可见的祖先；只有 project-wide direct consumers 才自然提升到 root。
获取可能需要更高层协调，但不会因此把所有发现和链接都提升到同一层。

消费优先使用依赖提供的 imported targets，如 `fmt::fmt`、`GTest::gtest`。只有依赖缺少现代 Target interface 时，才考虑自行组合 include paths、library paths 和 compiler flags。
链接可见性按 [Target 与接口](engineering.md#target-与接口) 判断。

## 依赖协调

FetchContent 的声明顺序、来源／版本覆盖和传递依赖具有 project hierarchy semantics，非 trivial 获取应由 dependency authority 统一协调，避免每个 component 随意独立声明和获取。
Target-local CMakeLists 负责消费已经满足的依赖。

`cmake/Dependencies.cmake` 不是默认必需文件。多个共享依赖、复杂 FetchContent、非 trivial 获取或协调逻辑明显挤占 root 时再引入，
只承担 project-level dependency coordination，不承担 Target 定义。简单的 `find_package(fmt CONFIG REQUIRED)` 可以直接放在合适 scope。

## 离线约束

仅在任务明确禁止网络或下载时，在首次 configure/build 前检查实际执行路径中的依赖获取入口，包括配置期的模块初始化和间接调用的获取脚本。
优先使用项目已有的离线开关或已提供的本地依赖，并核对其适用阶段；选择构建模块、关闭某个 Target 或命中本地缓存，都不能单独证明不会尝试下载。
确认相关入口已关闭或使用明确提供的本地输入，再执行配置及构建。必要依赖无法离线满足、或获取路径尚无法确认时，停止该执行路径并报告原因，不通过可能联网的 configure 试错，也不擅自关闭必需功能。
保留从首次尝试开始的命令和下载记录；若曾触发下载，分别报告执行偏差与修复后的复验结果，清理缓存不能撤销或隐去此前发生的获取动作。

## 可选依赖

依赖服从 feature ownership：功能关闭时不获取、不发现其依赖，也不创建对应 Target；不要先在 root REQUIRED 查找所有可能依赖再判断开关。
测试、benchmark 和 example 同样处理。

开启功能时需要完整的发现、消费、测试运行环境和适用的交付接线；可选动态依赖也应按实际 imported target/SDK 类型处理正式运行时部署。
SDK 不可用时保留诚实的缺包失败，不擅自关闭功能，并明确尚未验证的开启路径。

## Recipe 与版本选择

具名库先按 [recipe 查找与适用](recipes.md#查找与适用) 处理；Target ownership 决定其要求在当前工程中的接线位置。
具体库的集成、部署和 workaround 保留在 recipe；通用交付规则见 [交付](delivery.md)。

### 版本选择

- 明确升级、降级或指定版本时，当前任务在授权范围内替代已有 pin。
- 没有明确换版要求时保留 existing pin，新增 test target 或 REDESIGN 均不自动授权换版。新增获取优先使用适用 recipe 中用户指定的默认版本。
- 获取方式可在授权范围调整，并保留用户指定的兼容面和现实环境约束。
- 兼容约束始终是候选版本必须满足的条件；冲突时报告，不静默换版或换获取方式。
- 用户默认版本不是 upstream latest 声明。用户要求 latest/current stable 时核对官方资料；缺少版本不等于自动追 latest。
- 无 recipe 或缺少版本时，仅为当前工程需要查询资料；不把自动选择的版本持久化为用户默认值。离线或无法验证时如实报告。

## 验证

- 明确禁止下载：检查首次 configure/build 及后续复验的获取记录；缺失记录或缓存命中不能作为未访问网络的证明。
- 发现方式改变：用实际相关配置完成 configure/generate，确认预期 Target 可用；不能据此声称编译、链接或版本兼容已通过。
- 消费或传播关系改变：按 [工程基础的验证要求](engineering.md#验证) 构建受影响 Target 和 consumer。
- 可选功能改变：验证 OFF 不获取、发现或创建该功能；涉及 ON 的改动还需验证开启路径，OFF 成功不能代替 ON。环境不足时按入口要求报告阻塞。
- 影响安装或运行时交付时，执行 [交付](delivery.md#验证) 和适用的 [运行时验证](runtime.md#验证)。
