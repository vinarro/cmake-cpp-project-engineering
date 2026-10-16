# CMake C++ Project Engineering

规范 AI 编写 CMake C++ 工程的 Skill，统一目录组织、依赖管理与交付约定，提升跨项目一致性，保留设计灵活性。

## 用途

本项目将个人工程偏好整理为可复用的约束，使模型在不同项目、不同对话中保持一致的设计方向。模型根据当前任务、现有项目策略、平台和兼容要求选择具体实现。

适用于：

- 创建、修改或重新设计 CMake C++ 工程的构建与交付结构。
- 确定 Target 职责、源码归属、目录组织、接口传播及第三方依赖接线。
- 处理应用与 SDK 的安装、导出和运行时交付。
- 保存、更新、删除或应用用户指定的依赖 recipe。

普通 C++ 功能实现、测试用例编写和编辑器配置，如果不涉及构建接线，则不属于本 Skill 的范围。

其中的目录、命名、安装布局等默认值体现本项目选定的工程偏好，使用时按 Skill 的优先级处理；它们不代表所有 CMake 项目都必须采用的规范。

## 项目结构

```text
cmake-cpp-project-engineering/
├── README.md
├── LICENSE
├── .gitignore
├── .gitattributes
├── docs/
│   └── design.md
├── benchmarks/
└── skills/
    └── cmake-cpp-project-engineering/
        ├── SKILL.md
        ├── agents/
        │   └── openai.yaml
        └── references/
            ├── engineering.md
            ├── dependencies.md
            ├── delivery.md
            ├── runtime.md
            ├── recipes.md
            └── windows-shared-sdk.md
```

实际使用的 Skill 是 [`skills/cmake-cpp-project-engineering/`](skills/cmake-cpp-project-engineering/SKILL.md) 整个目录。模型需要的规则和参考文件均位于其中，可独立复制或链接使用。

[`docs/design.md`](docs/design.md) 面向项目维护者，记录设计目的与内容取舍原则。仓库外层的文档和 benchmark 不属于 Skill 的运行依赖。

`benchmarks/` 预留给后续有价值的验证任务，当前为空。

## 安装与使用

将 `skills/cmake-cpp-project-engineering/` 整体复制或链接到所用工具识别的 Skill 目录，保留目录内部结构。仅克隆源码仓库不会自动启用 Skill。

Codex 的发现位置和链接方式以[官方 Skill 文档](https://learn.chatgpt.com/docs/build-skills)为准。也可以向 Codex 的安装器发出明确请求：

```text
使用 $skill-installer，从仓库 https://github.com/vinarro/cmake-cpp-project-engineering
安装 skills/cmake-cpp-project-engineering 目录中的 Skill。
```

启用后，可以在 Codex 中明确指定 Skill，例如：

```text
使用 $cmake-cpp-project-engineering，整理当前项目中 core 组件的 CMake 表达，保持公开接口和产物不变。
```

```text
使用 $cmake-cpp-project-engineering，为现有源码建立 CMake 构建，交付一个静态 SDK，并验证安装后的独立消费。
```

## 用户依赖 recipe

依赖 recipe 的默认位置为 Skill 内的 `references/dependency-recipes/`，也可由用户指定其他位置；未保存任何条目时目录可以为空或不存在。具体约定见 [recipe 说明](skills/cmake-cpp-project-engineering/references/recipes.md)。

recipe 记录用户指定的依赖策略，更新 Skill 时应保留已有条目。

## 维护与许可

修改原则见 [设计说明](docs/design.md)。

本项目采用 [MIT License](LICENSE)。
