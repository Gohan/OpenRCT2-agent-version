# AI代理实施指南 - 概要

## 概述

本仓库包含一份全面的指南（**游戏AI代理化实施指南.md**），详细说明如何在经营管理/模拟类游戏中实现AI代理，该指南基于 OpenRCT2-agent-version 项目编写。

该指南为希望在游戏中集成AI助手（如 Claude Code）以增强游戏性和交互体验的游戏开发者提供详细的技术指导。

## 本项目的功能

OpenRCT2-agent-version 是 OpenRCT2 的一个实验性分支，集成了AI能力：

1. **游戏内终端** - 嵌入游戏窗口的完整 VT100/xterm 终端
2. **自动启动 Claude Code** - 终端打开时自动启动 Claude Code AI助手
3. **自定义 CLI 工具 (rctctl)** - kubectl 风格的命令行界面用于游戏控制
4. **JSON-RPC API 服务器** - 通过 JSON-RPC 2.0 协议暴露游戏状态和操作（端口 9876）

## 核心创新

- **AI作为协同玩家**：AI 使用与人类玩家相同的规则玩游戏（不作弊）
- **CLI驱动**：AI 通过命令交互，而非视觉界面
- **深度集成**：Fork 方式允许 PTY、终端仿真和原生 UI
- **可扩展架构**：通信层、CLI 和游戏逻辑之间清晰分离

## 核心组件（约 15,000 行新代码）

| 组件 | 文件数 | 代码行数 | 说明 |
|------|-------|---------|------|
| JSON-RPC 服务器 | 2 | ~600 | 游戏通信的 TCP 服务器 |
| RPC 处理器 | 15 | ~8,000 | 公园、游乐设施、员工、游客等业务逻辑 |
| 终端系统 | 10 | ~4,200 | PTY、libvterm 集成、UI 窗口 |
| rctctl CLI 工具 | 30+ | ~5,000 | 独立的命令行界面 |
| 测试 | 5 | ~800 | Python 测试脚本 |

## 指南内容（游戏AI代理化实施指南.md）

全面指南涵盖：

1. **项目背景** - 改动内容和原因
2. **架构分析** - 详细的系统设计和数据流
3. **核心实现** - 各组件深度解析
4. **通用实施步骤** - 如何在您的游戏中实现
5. **最佳实践** - API 设计、错误处理、性能、安全性
6. **平台支持** - Windows、macOS、Linux 注意事项
7. **常见问题** - 常见问题和解决方案
8. **附录** - 完整的 API 参考、模板、资源

## 其他游戏快速入门

该指南提供了实现类似功能的分步流程：

### 第1阶段：MVP（第1-3周）
- 实现 JSON-RPC 服务器（或类似通信层）
- 创建基础 CLI 工具
- 暴露 5-10 个核心 API
- 使用外部终端测试

### 第2阶段：增强（第4-6周）
- 扩展到 10-15 个 API
- 添加错误处理和验证
- 编写 AI 系统提示词
- 性能优化

### 第3阶段：完善（第7-8周，可选）
- 集成游戏内终端
- 自动启动 AI 代理
- 添加 UI 控制
- 会话日志记录

## 技术栈

- **语言**：C++（游戏）、C++（CLI 工具）
- **通信**：JSON-RPC 2.0 over TCP sockets
- **终端**：libvterm + PTY（Unix 上用 forkpty，Windows 上用 ConPTY）
- **JSON**：nlohmann/json
- **构建**：CMake
- **测试**：Python + CTest

## 适用的游戏类型

**高度适合**：
- ⭐⭐⭐⭐⭐ 经营管理/大亨游戏（模拟城市、过山车大亨2）
- ⭐⭐⭐⭐⭐ 策略游戏（文明、全面战争）
- ⭐⭐⭐⭐ 建造/生产游戏（异星工厂、幸福工厂）
- ⭐⭐⭐⭐ 模拟游戏（农场模拟器、游戏开发大亨）

**不适合**：
- ❌ 实时动作游戏（需要毫秒级反应）
- ❌ 在线竞技游戏（可能被视为作弊）

## 设计哲学

**核心原则**：AI 应该像玩家一样玩，而不是作弊

- ✅ AI 使用相同的游戏规则和操作
- ✅ AI 通过 CLI 查询结构化数据（无法"看到"屏幕）
- ✅ AI 在公平的限制下运作
- ❌ 不直接操作内存
- ❌ 不访问隐藏信息
- ❌ 不使用调试/作弊命令（除非玩家也能用）

## 架构图

```
┌─────────────────────────────────────────┐
│         OpenRCT2 游戏进程                │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ AI 代理终端窗口                   │  │
│  │  • libvterm（VT100 仿真）         │  │
│  │  • PTY（伪终端）                  │  │
│  └───────────┬───────────────────────┘  │
│              │ 生成                     │
│  ┌───────────▼───────────────────────┐  │
│  │ Claude Code 进程                  │  │
│  │  • cwd: ai-agent-workspace/       │  │
│  │  • PATH 包含 rctctl               │  │
│  └───────────┬───────────────────────┘  │
│              │ 调用                     │
│  ┌───────────▼───────────────────────┐  │
│  │ JSON-RPC 服务器 (localhost:9876)  │  │
│  └───────────┬───────────────────────┘  │
│              │ 分发                     │
│  ┌───────────▼───────────────────────┐  │
│  │ Handler Registry                  │  │
│  │  • park.*, ride.*, staff.*, ...   │  │
│  └───────────┬───────────────────────┘  │
│              │ 访问                     │
│  ┌───────────▼───────────────────────┐  │
│  │ 游戏状态 & 操作                   │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

外部工具:
┌──────────┐      ┌──────────────┐
│  rctctl  │─TCP─▶│  JSON-RPC    │
│ (CLI工具) │      │  :9876       │
└──────────┘      └──────────────┘
```

## 示例：查询公园状态

```bash
# 人类可读输出
$ rctctl park status
Park: Gohan's Park
Cash: $12,500.00
Rating: 678
Guests: 543
Date: Year 3, September 15

# 机器可读输出（供 AI 解析）
$ rctctl park status -o json
{
  "name": "Gohan's Park",
  "cash": 12500.00,
  "rating": 678,
  "guests": 543,
  "date": {
    "year": 3,
    "month": 9,
    "day": 15
  }
}
```

## 关键文件

### 文档
- **游戏AI代理化实施指南.md** - 全面实施指南（中文，1100+ 行）
- **CODING_AGENT.md** - 架构概述
- **AGENTS.md** - 项目背景
- **RCTCTL.md** - CLI 设计模式

### 实现
- `src/openrct2/scripting/JsonRpcServer.{h,cpp}` - RPC 服务器
- `src/openrct2/scripting/rpc/handlers/` - API 处理器（13 个文件）
- `src/openrct2/terminal/` - 终端子系统（6 个文件）
- `src/openrct2-ui/windows/AIAgentTerminal.cpp` - 终端 UI（3220 行！）
- `rctctl/` - CLI 工具（独立项目）

### AI 工作区
- `ai-agent-workspace/IN_GAME_AGENT.md` - AI 系统提示词（约 500 行）
- `ai-agent-workspace/auto_prompts.txt` - 自动提示词轮换

## 构建

```bash
# 安装依赖（macOS）
brew install libvterm nlohmann-json

# 配置
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release

# 构建所有内容（游戏 + CLI + 资源）
cmake --build build --target agent_bundle -j8

# 运行
./build/OpenRCT2.app/Contents/MacOS/OpenRCT2
```

## 测试

```bash
# CLI 验证测试（快速，不需要游戏）
ctest -R rctctl_validation

# 端到端测试（使用无头游戏）
ctest -R agent_scenarios
```

## 许可证

GNU General Public License v3.0（与 OpenRCT2 相同）

## 致谢

- 基于 [OpenRCT2](https://github.com/OpenRCT2/OpenRCT2)
- 灵感来自"AI 玩 Pokemon"实验
- 为与 [Claude Code](https://claude.ai) 配合使用而构建

## 贡献

这是一个实验性分支。欢迎贡献，特别是：
- Windows ConPTY 支持
- 额外的 RPC 处理器
- 改进的 AI 提示词
- AI 会话的 bug 报告

---

**完整的技术指南包含实现细节、最佳实践和您自己游戏的分步说明，请参阅：游戏AI代理化实施指南.md**

