---
name: cmake-cpp-project-engineering
description: 创建、修改或重设计 CMake C++ 构建与交付结构；保存、更新或删除用户依赖 recipe。不用于不涉及构建接线的普通 C++ 实现、测试用例编写或编辑器配置。
---

# CMake C++ Project Engineering

按用户选定的工程策略推导 Target responsibilities、Target graph、source/dependency ownership，再表达为 CMake、交付和验证。
参考文档是用户选定的有效工程策略；其中应用目录、项目内 install prefix 等是个人默认值，不宣称为 CMake 通用规范。
CMake 表达应清晰、与项目风格一致，Target 的职责、源码及依赖关系易于定位和追踪。

## 任务入口与操作范围

先区分工程任务与 recipe 维护。纯 recipe 维护不要求存在 C++ 工程，也不触发 configure/build。
一个请求同时包含两者时，分别处理各自授权范围；保存默认策略不自动修改现有项目。

| Operation | 判定与应用范围 |
|---|---|
| CREATE | 建立尚不存在的 CMake 工程，包括为已有源码新建构建；以推荐策略推导目标工程 |
| CHANGE | 已有工程上的修改、修复或局部重构；只改变完成任务所需范围 |
| REDESIGN | 明确要求全面重新设计、不保留旧 CMake 结构；在授权范围重新推导工程 |

“整理”“优化”“现代化”“按推荐方式整理”默认是 CHANGE；只有明确授权全面重设计才进入 REDESIGN。
CHANGE 按任务要求整理或修复受影响范围，不顺带统一无关目录、版本、Target 名或依赖策略。
REDESIGN 可以只针对指定子系统，授权不扩大到整个仓库。旧 CMake 用于恢复语义，不约束新组织方式；
可重新考虑 minimum version、Target 拆分和命名、目录、依赖接线、Presets、CMakeLists 边界、usage requirements 和 install/export。
CREATE/REDESIGN 以推荐结构和实际工程需求为质量标准，不以“能编译的最少文件”作为完成标准。

## 适用优先级

先按当前显式用户要求和任务授权范围确定必须满足的条件。除明确要求改变的部分外，保留必要功能、源码、生成步骤、依赖、产物、平台行为及兼容契约。
现实环境约束始终需要核对；条件冲突时报告，不能靠忽略约束声称完成。
未明确要求换版时保留已有依赖版本；结构重设计不自动授权换版，具体选择见 [依赖](references/dependencies.md#版本选择)。

只在满足上述条件的方案间，按下表选择设计偏好：

| Operation | 设计偏好（从高到低） |
|---|---|
| CREATE / REDESIGN | 用户 recipes/defaults → 推荐策略 |
| CHANGE | 现有有意的项目策略 → 用户 recipes/defaults → 受影响范围内的推荐策略 |

## 阅读路径

CREATE 和 REDESIGN 先读 [核心决策](references/engineering.md#核心决策)，再按下表读取受影响的章节，不要求通读工程基础。
创建或重建项目级配置时，还需读取 [语言与配置](references/engineering.md#语言与配置) 和 [Presets](references/engineering.md#presets)。
CREATE 新建产品应用时默认建立正式安装交付，须读取 [交付](references/delivery.md)。REDESIGN 在授权范围包含产品应用交付时应用该默认值；整个产品工程的重新设计包含这一范围，局部组件重设计不自动包含。
用户明确选择外部打包或其他交付方式时遵循该方式，并读取适用的交付规则。
REDESIGN 只应用获授权范围内的规则；CHANGE 只读受影响章节。各专题按条件读取小节及末尾的相关验证要求，不连带加载不适用内容。
纯 recipe 维护只读 [recipe](references/recipes.md) 及指定条目，不加载工程实现和交付专题。

| 本次任务涉及 | 读取 |
|---|---|
| Target 职责、拆分或组合 | [核心决策](references/engineering.md#核心决策) |
| Target 接口或依赖传播 | [Target 与接口](references/engineering.md#target-与接口) |
| 源码声明、Target 命名 | [源码声明](references/engineering.md#source-declaration)、[组件命名](references/engineering.md#component-naming) |
| 公开头文件契约 | [Public Headers](references/engineering.md#public-headers) |
| Windows shared library 或 MODULE 插件导出 | [DLL 与插件导出](references/engineering.md#dll-与插件导出) |
| 目录、CMakeLists 边界、资源或生成文件 | [目录与 Ownership](references/engineering.md#目录与-ownership) |
| baseline、语言或配置 scope | [语言与配置](references/engineering.md#语言与配置) |
| Presets | [Presets](references/engineering.md#presets) |
| tests/benchmarks/examples 或 subproject 复用 | [测试与子项目](references/engineering.md#测试与子项目) |
| 第三方依赖的获取、发现、消费、版本或可选功能 | [依赖](references/dependencies.md)；具名库先按 [recipe 查找与适用](references/recipes.md#查找与适用) 处理 |
| 明确禁止网络或下载，且将运行 configure/build | 先读 [离线约束](references/dependencies.md#离线约束)，在首次执行前检查获取入口 |
| 产品应用或 SDK 的安装、布局、package/export | [交付](references/delivery.md)，按应用或 SDK 阅读对应章节 |
| 动态运行时、framework 部署、安装脚本 | [运行时](references/runtime.md)；目录决策来自交付专题 |
| recipe 的新建、更新、删除或应用 | [recipe](references/recipes.md) |

只为当前任务加载规则和匹配的 recipe，不把整个 skill 再写入项目。参考文档路径相对本 Skill；文中的项目示例路径按实际工程替换。

## 验证与交付

按本次改动影响的构建、依赖消费、测试接线或交付行为，选择相关专题末尾的验证要求，优先使用项目已有入口。
CHANGE 按影响范围验证；REDESIGN 还须覆盖需要保留的产物、消费者、平台接线和兼容面。
测试相关工作聚焦测试 Target 的构建、CTest 注册、配置和运行环境，复用现有测试内容；用户明确要求的功能验证按任务要求执行。
注释或格式修改先检查 diff，再按语义风险决定是否配置。环境可用且任务未禁止时完成相关验证并处理失败。

任务涉及交付时，从干净输出目录按本次任务认可的正式流程重建：使用 CMake 安装时验证 fresh install；
用户明确选择外部打包或手工交付流程时验证该流程。流程之外的临时补救不能作为交付完成的证据。

验证失败或环境受阻时，说明失败阶段、已知原因和未验证范围，不为通过检查私自换版、切换包管理器、移除测试或降低用户约束。
没有可行验证路径时停止该路径；修改前已无法配置时，可继续合理的静态修改，但不能据此声称相关行为已经验证。

交付说明实际改动范围、重要兼容变化、采用的 recipe/版本来源及相关验证结果；保留必要的命令、退出码和失败摘要，区分通过、失败、未执行和环境阻塞。
报告实际使用的 CMake 版本、平台和配置；未覆盖的 minimum 版本或其他平台、配置列为未验证。
不要求固定长报告、计划文件、逐步审批或通用 TDD/Git 工作流。
