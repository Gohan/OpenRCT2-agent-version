# 游戏AI代理化实施指南

基于 OpenRCT2-agent-version 项目的完整技术指南

---

## 📋 目录

1. [项目背景与创新](#1-项目背景与创新)
2. [核心改动分析](#2-核心改动分析)  
3. [架构设计](#3-架构设计)
4. [核心实现详解](#4-核心实现详解)
5. [通用实施步骤](#5-通用实施步骤)
6. [最佳实践](#6-最佳实践)
7. [平台支持](#7-平台支持)
8. [常见问题FAQ](#8-常见问题faq)
9. [总结与展望](#9-总结与展望)
10. [附录](#10-附录)

---

## 1. 项目背景与创新

### 1.1 原始游戏

**OpenRCT2** = RollerCoaster Tycoon 2 的开源重新实现
- 经典经营管理模拟游戏
- C++ 重写，支持现代OS
- 活跃的开源社区

### 1.2 AI代理版本做了什么？

这是一个**疯狂的实验**：让 AI 编程助手（Claude Code）直接在游戏里玩游戏！

**四大核心创新：**

1. **游戏内嵌终端**
   - 真实的 VT100/xterm 终端
   - libvterm + PTY 实现
   - 256色 + RGB 支持

2. **自动启动 Claude Code**
   - 打开终端即启动 AI
   - 专用工作区配置
   - 自动配置 PATH

3. **专用 CLI 工具 (rctctl)**
   - 类 kubectl 风格
   - 人类可读 + JSON 输出
   - 完整 --help 文档

4. **JSON-RPC API 服务器**
   - localhost:9876
   - JSON-RPC 2.0 协议
   - 13+ 业务处理器

### 1.3 设计哲学：AI 作为协同玩家

**核心理念**：AI 应该像玩家一样玩，而不是作弊

| 原则 | 说明 |
|------|------|
| ✅ 相同规则 | AI 通过游戏 Action 系统执行操作 |
| ✅ 公平限制 | AI 无法看到画面，只能用 CLI 查询 |
| ✅ 真实挑战 | AI 需要"学习"游戏机制 |
| ❌ 禁止作弊 | 不能直接修改内存或变量 |
| ❌ 禁止透视 | 不能访问隐藏信息 |

**就像**：盲人通过助手描述玩游戏，或通过API玩股票交易

---

## 2. 核心改动分析

### 2.1 改动规模

相比原版 OpenRCT2，新增 **约15,000行代码**：

| 组件 | 文件 | 代码 | 说明 |
|------|-----|------|------|
| JSON-RPC 服务器 | 2 | ~600 | 核心通信 |
| RPC 处理器 | 15 | ~8,000 | 业务逻辑 |
| 终端子系统 | 10 | ~4,200 | PTY+UI |
| rctctl CLI | 30+ | ~5,000 | 独立工具 |
| 测试 | 5 | ~800 | Python测试 |
| 文档 | 5 | ~3,000 | 指南 |

### 2.2 关键文件修改

**新增核心文件**：
```
src/openrct2/scripting/
├── JsonRpcServer.{h,cpp}           # TCP服务器
└── rpc/
    ├── HandlerRegistry.{h,cpp}     # 方法分发
    ├── RpcTypes.h                  # 类型定义
    ├── RpcUtils.{h,cpp}            # 工具函数
    └── handlers/
        ├── ParkHandlers.cpp        # 公园
        ├── RideHandlers.cpp        # 设施
        ├── StaffHandlers.cpp       # 员工
        ├── GuestHandlers.cpp       # 游客
        ├── FinanceHandlers.cpp     # 财务
        └── ... (13个处理器)

src/openrct2/terminal/
├── ShellProcess.{h,cpp}            # PTY抽象
├── TerminalSession.{h,cpp}         # libvterm
├── AIAgentLaunch.{h,cpp}           # Claude启动
├── SessionFileMonitor.{h,cpp}      # 回合检测
└── SessionLogGenerator.{h,cpp}     # 日志

src/openrct2-ui/windows/
└── AIAgentTerminal.cpp             # 终端UI(3220行!)

rctctl/                             # 独立CLI项目
├── src/
│   ├── main.cpp
│   ├── cli/                        # 解析
│   ├── commands/                   # 命令
│   ├── renderers/                  # 输出
│   └── rpc/                        # RPC客户端
└── CMakeLists.txt

ai-agent-workspace/
├── IN_GAME_AGENT.md                # AI提示词(~500行)
└── auto_prompts.txt                # 自动提示
```

**修改的文件**：
- `CMakeLists.txt` - 添加 agent_bundle 目标
- `ScriptEngine.cpp` - 启动 RPC 服务器
- `WindowClass` 枚举 - 新增 aiAgentTerminal

---

## 3. 架构设计

### 3.1 整体架构图

```
┌──────────────────────────────────────────────────────┐
│              OpenRCT2 游戏进程                        │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │ AI Agent 终端窗口                               │ │
│  │  • libvterm (VT100/xterm模拟)                  │ │
│  │  • PTY (伪终端)                                 │ │
│  │  • ANSI 256色                                   │ │
│  └───────────┬──────────────────────────────────────┘ │
│              │ fork/exec                              │
│  ┌───────────▼──────────────────────────────────────┐ │
│  │ Claude Code 子进程                               │ │
│  │  • cwd: ai-agent-workspace/                     │ │
│  │  • PATH += rctctl                               │ │
│  │  • 读取 IN_GAME_AGENT.md                        │ │
│  └───────────┬──────────────────────────────────────┘ │
│              │ 调用                                   │
│  ┌───────────▼──────────────────────────────────────┐ │
│  │ JSON-RPC Server                                  │ │
│  │  • TCP localhost:9876                           │ │
│  │  • 换行分隔JSON消息                              │ │
│  │  • 非阻塞I/O                                     │ │
│  └───────────┬──────────────────────────────────────┘ │
│              │ 调度                                   │
│  ┌───────────▼──────────────────────────────────────┐ │
│  │ Handler Registry                                 │ │
│  │  • 方法名 → Handler函数                          │ │
│  │  • park.*, ride.*, staff.*, ...                 │ │
│  └───────────┬──────────────────────────────────────┘ │
│              │ 访问                                   │
│  ┌───────────▼──────────────────────────────────────┐ │
│  │ Game State & Actions                             │ │
│  │  • getGameState()                               │ │
│  │  • GameActions::Execute()                       │ │
│  │  • RideManager, ParkData, ...                   │ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘

外部进程:
┌─────────────┐      ┌────────────────┐
│   rctctl    │─TCP─▶│  JSON-RPC      │
│  (CLI工具)  │      │  :9876         │
└─────────────┘      └────────────────┘
```

### 3.2 数据流示例

**场景**: AI 想要查询公园状态

```
1. Claude 执行: rctctl park status

2. rctctl 构造请求:
   {
     "jsonrpc": "2.0",
     "method": "park.status",
     "params": {},
     "id": 1
   }

3. TCP 发送到 localhost:9876

4. JsonRpcServer 接收并解析

5. HandlerRegistry 分发到 ParkHandlers

6. ParkHandlers 访问 gameState.park:
   - name
   - cash
   - rating
   - guests
   - date
   - ...

7. 构造响应:
   {
     "jsonrpc": "2.0",
     "result": {
       "name": "Gohan's Park",
       "cash": 12500.00,
       "rating": 678,
       "guests": 543,
       ...
     },
     "id": 1
   }

8. rctctl 渲染表格:
   Park: Gohan's Park
   Cash: $12,500.00
   Rating: 678
   Guests: 543
   ...

9. Claude 读取输出并做决策
```

### 3.3 关键技术决策

#### 为什么 JSON-RPC 2.0？

| 优点 | 说明 |
|------|------|
| ✅ 标准化 | 广泛支持，LLM 友好 |
| ✅ 简单 | 请求-响应，无状态 |
| ✅ 可扩展 | 轻松添加新方法 |
| ✅ 易调试 | 文本协议，可用 netcat 测试 |

#### 为什么独立 CLI 工具？

| 优点 | 说明 |
|------|------|
| ✅ 解耦 | 游戏和CLI分离 |
| ✅ 复用 | 脚本、测试、其他AI都能用 |
| ✅ 一致性 | C++（与游戏一致） |
| ✅ 多输出 | 表格（人类）+ JSON（程序） |

#### 为什么 Fork 而非插件？

| Fork | 插件 |
|------|------|
| ✅ 深度集成（PTY、渲染） | ❌ 受限于插件API |
| ✅ 原生性能 | ⚠️ 性能开销 |
| ❌ 与上游分离 | ✅ 跟随上游更新 |
| ❌ 长期维护难 | ✅ 维护简单 |

**本项目选择 Fork** 因为终端需要 PTY、SDL 输入、渲染管线等深度集成。

#### 为什么 PTY + libvterm？

| 组件 | 作用 |
|------|------|
| PTY | 提供伪终端设备，支持交互式程序 |
| libvterm | 解析 ANSI 转义序列（颜色、光标移动等） |

这样才能运行真实的 shell 程序（如 Claude Code、bash、vim 等）

---

## 4. 核心实现详解

### 4.1 JSON-RPC 服务器

**文件**: `src/openrct2/scripting/JsonRpcServer.{h,cpp}`

**核心类**:
```cpp
class JsonRpcServer {
public:
    JsonRpcServer();
    ~JsonRpcServer();
    
    // 启动服务器
    bool Start(int32_t port = 9876);
    
    // 停止服务器
    void Stop();
    
    // 每帧调用（游戏主循环）
    void Update();
    
private:
    // TCP 监听socket
    std::unique_ptr<ITcpSocket> _listener;
    
    // 已连接客户端
    std::vector<ClientConnection> _clients;
    
    // 方法注册表
    HandlerRegistry _registry;
    
    // 接受新连接
    void AcceptConnections();
    
    // 处理客户端数据
    void ProcessClientData(ClientConnection& client);
    
    // 分发 RPC 方法
    RpcResult DispatchMethod(
        const std::string& method,
        const json& params
    );
};
```

**初始化流程**:
```cpp
// 在 ScriptEngine::Initialise() 中
void ScriptEngine::Initialise() {
    // ... 现有初始化 ...
    
    #ifdef ENABLE_SCRIPTING
    // 启动 JSON-RPC 服务器
    _rpcServer = std::make_unique<JsonRpcServer>();
    if (!_rpcServer->Start()) {
        LOG_ERROR("Failed to start JSON-RPC server");
    }
    #endif
}
```

**协议示例**:
```jsonc
// 请求
{
  "jsonrpc": "2.0",
  "method": "park.status",
  "params": {},
  "id": 1
}

// 成功响应
{
  "jsonrpc": "2.0",
  "result": {
    "name": "My Park",
    "cash": 10000.00,
    "rating": 678
  },
  "id": 1
}

// 错误响应
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32601,
    "message": "Method not found"
  },
  "id": 1
}
```

### 4.2 Handler Registry

**文件**: `src/openrct2/scripting/rpc/HandlerRegistry.{h,cpp}`

**核心类**:
```cpp
class HandlerRegistry {
public:
    using HandlerFunc = std::function<RpcResult(const json&)>;
    
    // 注册方法
    void Register(
        const std::string& method,
        HandlerFunc handler
    );
    
    // 调用方法
    RpcResult Call(
        const std::string& method,
        const json& params
    );
    
    // 检查方法是否存在
    bool Has(const std::string& method) const;
    
private:
    std::unordered_map<std::string, HandlerFunc> _handlers;
};
```

**注册模式**:
```cpp
// handlers/ParkHandlers.cpp
void RegisterParkHandlers(HandlerRegistry& registry) {
    // 查询类
    registry.Register("park.status", [](const json& params) {
        auto& gs = GetGameState();
        json result;
        result["name"] = gs.park.Name;
        result["cash"] = MoneyToDouble(gs.park.Cash);
        result["rating"] = gs.park.Rating;
        result["guests"] = gs.park.NumGuests;
        return result;
    });
    
    // 操作类
    registry.Register("park.setEntranceFee", [](const json& params) {
        // 参数验证
        if (!params.contains("fee")) {
            return RpcError(-32602, "Missing required param: fee");
        }
        
        // 转换金额
        auto fee = DoubleToMoney(params["fee"].get<double>());
        
        // 执行 Action
        auto action = ParkSetEntranceFeeAction(fee);
        auto result = GameActions::Execute(&action);
        
        // 检查结果
        if (result.Error != GameActions::Status::Ok) {
            return RpcError(-32001, 
                BuildGameActionErrorMessage(result));
        }
        
        return BuildActionSuccessPayload();
    });
}
```

**初始化**:
```cpp
// HandlerRegistry.cpp
void HandlerRegistry::InitializeAll() {
    RegisterParkHandlers(*this);
    RegisterRideHandlers(*this);
    RegisterStaffHandlers(*this);
    RegisterGuestHandlers(*this);
    RegisterFinanceHandlers(*this);
    RegisterMapHandlers(*this);
    RegisterShopHandlers(*this);
    RegisterResearchHandlers(*this);
    RegisterNewsHandlers(*this);
    RegisterWeatherHandlers(*this);
    RegisterWindowHandlers(*this);
    RegisterAgentHandlers(*this);
}
```


### 4.3 终端集成

**核心文件**: `src/openrct2/terminal/TerminalSession.{h,cpp}`

**TerminalSession 类**:
```cpp
class TerminalSession {
public:
    TerminalSession(int rows, int cols);
    ~TerminalSession();
    
    // 写入数据到 shell
    void Write(const std::string& data);
    
    // 从 shell 读取（非阻塞）
    std::string Read();
    
    // 获取渲染单元格
    const std::vector<TerminalCell>& GetCells() const;
    
    // 调整大小
    void Resize(int rows, int cols);
    
    // 获取光标位置
    VTermPos GetCursorPos() const;
    
private:
    VTerm* _vterm;              // libvterm 实例
    VTermScreen* _screen;       // 屏幕缓冲区
    ShellProcess _shell;        // PTY 抽象
    std::vector<TerminalCell> _cells;  // 渲染缓存
    
    // libvterm 回调
    static int OnDamage(VTermRect rect, void* user);
    static int OnMoveCursor(VTermPos pos, void* user);
};
```

**TerminalCell 结构**:
```cpp
struct TerminalCell {
    char32_t codepoint;  // Unicode 字符
    VTermColor fg;       // 前景色 (RGB)
    VTermColor bg;       // 背景色 (RGB)
    struct {
        uint8_t bold : 1;
        uint8_t underline : 1;
        uint8_t italic : 1;
        uint8_t blink : 1;
        uint8_t reverse : 1;
    } attrs;
};
```

**PTY 抽象** (`ShellProcess.{h,cpp}`):
```cpp
class ShellProcess {
public:
    // 启动 shell
    bool Start(const std::vector<std::string>& cmd,
               const std::map<std::string, std::string>& env);
    
    // 写入数据
    ssize_t Write(const void* data, size_t len);
    
    // 读取数据（非阻塞）
    ssize_t Read(void* buffer, size_t len);
    
    // 获取 PTY 主端 FD
    int GetMasterFd() const { return _masterFd; }
    
    // 检查进程是否还活着
    bool IsAlive() const;
    
private:
    pid_t _pid = -1;
    int _masterFd = -1;
    
    // macOS/Linux: 使用 forkpty()
    // Windows: 需要 ConPTY (未实现)
};
```

### 4.4 AI 代理启动

**文件**: `src/openrct2/terminal/AIAgentLaunch.{h,cpp}`

**核心函数**:
```cpp
// 查找可执行文件
std::optional<std::string> FindExecutable(const std::string& name) {
    // 在 PATH 中搜索
    const char* pathEnv = std::getenv("PATH");
    // ... 解析和搜索逻辑 ...
}

// 准备 Claude Code 启动
std::optional<std::vector<std::string>> 
PrepareClaudeCodeLaunch(const std::string& parkName) {
    // 1. 查找 claude 可执行文件
    auto claudePath = FindExecutable("claude");
    if (!claudePath) {
        return std::nullopt;
    }
    
    // 2. 准备工作区
    auto workspaceDir = SetupAgentWorkspace(parkName);
    
    // 3. 构建命令
    std::vector<std::string> cmd = {
        *claudePath,
        "--dangerously-skip-permissions",  // 跳过安全确认
        "--working-directory", workspaceDir
    };
    
    return cmd;
}

// 设置工作区
std::string SetupAgentWorkspace(const std::string& parkName) {
    // 1. 查找或创建工作区目录
    auto workspaceDir = FindOrCreateWorkspace();
    
    // 2. 复制 IN_GAME_AGENT.md（如果不存在）
    CopyAgentInstructions(workspaceDir);
    
    // 3. 设置环境变量（rctctl 路径等）
    SetupEnvironment();
    
    return workspaceDir;
}
```

**环境设置**:
```cpp
std::map<std::string, std::string> GetAgentEnvironment() {
    std::map<std::string, std::string> env;
    
    // 添加 rctctl 到 PATH
    std::string path = std::getenv("PATH");
    path = GetRctctlDirectory() + ":" + path;
    env["PATH"] = path;
    
    // 设置 TERM
    env["TERM"] = "xterm-256color";
    env["COLORTERM"] = "truecolor";
    
    // 设置工作区
    env["AI_WORKSPACE"] = GetWorkspaceDirectory();
    
    return env;
}
```

### 4.5 rctctl CLI 工具

**文件**: `rctctl/src/main.cpp`

**主流程**:
```cpp
int main(int argc, char** argv) {
    try {
        // 1. 解析 CLI 参数
        auto cli = rctctl::cli::ParseCli(argc, argv);
        
        // 处理 --help
        if (cli.helpRequested) {
            if (cli.resourceHelpRequested) {
                rctctl::commands::PrintResourceUsage(cli.resource);
            } else {
                rctctl::commands::PrintUsage();
            }
            return 0;
        }
        
        // 2. 查找命令规范
        std::vector<std::string> commandPath = {cli.action};
        commandPath.insert(commandPath.end(), 
                          subcommands.begin(), 
                          subcommands.end());
        
        auto* spec = rctctl::commands::FindCommandSpec(
            cli.resource, 
            commandPath
        );
        
        if (!spec) {
            throw std::runtime_error("Unknown command");
        }
        
        // 3. 构建 RPC 调用计划
        auto parsedArgs = rctctl::cli::ParseCommandArguments(flags);
        auto plan = spec->buildPlan(parsedArgs);
        
        // 4. 执行 RPC 调用
        rctctl::rpc::JsonRpcClient client(cli.host, cli.port);
        auto rpcResult = client.Call(plan.method, plan.params);
        
        // 5. 渲染输出
        rctctl::renderers::RenderContext ctx{cli.outputFormat};
        spec->renderer(rpcResult, ctx);
        
        return 0;
        
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return 1;
    }
}
```

**命令注册示例**:
```cpp
// commands/park.cpp
CommandSpec parkStatusSpec = {
    .resource = "park",
    .action = "status",
    .description = "Get current park status",
    .usage = "rctctl park status",
    .rpcMethod = "park.status",
    
    .buildPlan = [](const Args& args) -> CommandPlan {
        return CommandPlan{
            .method = "park.status",
            .params = json::object()
        };
    },
    
    .renderer = [](const json& result, RenderContext& ctx) {
        if (ctx.format == "json") {
            std::cout << result.dump(2) << std::endl;
        } else {
            std::cout << "Park: " << result["name"] << "\n";
            std::cout << "Cash: $" << result["cash"] << "\n";
            std::cout << "Rating: " << result["rating"] << "\n";
            std::cout << "Guests: " << result["guests"] << "\n";
        }
    }
};
```

**JSON-RPC 客户端**:
```cpp
// rpc/json_rpc_client.cpp
class JsonRpcClient {
public:
    JsonRpcClient(const std::string& host, int port);
    
    json Call(const std::string& method, const json& params) {
        // 1. 连接到服务器
        if (!_socket || !IsConnected()) {
            Connect();
        }
        
        // 2. 构建请求
        json request = {
            {"jsonrpc", "2.0"},
            {"method", method},
            {"params", params},
            {"id", ++_nextId}
        };
        
        // 3. 发送请求（换行符分隔）
        std::string requestStr = request.dump() + "\n";
        Send(requestStr);
        
        // 4. 接收响应
        std::string responseStr = Receive();
        json response = json::parse(responseStr);
        
        // 5. 检查错误
        if (response.contains("error")) {
            throw RpcException(response["error"]);
        }
        
        return response["result"];
    }
    
private:
    int _socket = -1;
    int _nextId = 0;
};
```

### 4.6 UI 窗口

**文件**: `src/openrct2-ui/windows/AIAgentTerminal.cpp` (3220行!)

**核心功能**:

1. **渲染终端单元格**
```cpp
void RenderTerminalCells(DrawPixelInfo& dpi, 
                        const std::vector<TerminalCell>& cells,
                        int rows, int cols) {
    for (int row = 0; row < rows; ++row) {
        for (int col = 0; col < cols; ++col) {
            auto& cell = cells[row * cols + col];
            
            // 绘制背景
            GfxFillRect(dpi, 
                       col * charWidth, 
                       row * charHeight,
                       (col+1) * charWidth, 
                       (row+1) * charHeight,
                       ConvertVTermColor(cell.bg));
            
            // 绘制字符
            DrawCharacter(dpi, 
                         cell.codepoint,
                         col * charWidth,
                         row * charHeight,
                         ConvertVTermColor(cell.fg),
                         cell.attrs);
        }
    }
}
```

2. **处理输入**
```cpp
void OnKeyPress(const KeyEvent& event) {
    // 转换键盘事件为 VT100 序列
    std::string sequence = ConvertKeyToVT100(event);
    
    // 发送到终端会话
    _terminalSession->Write(sequence);
}
```

3. **更新循环**
```cpp
void Update() {
    // 从 shell 读取新数据
    std::string data = _terminalSession->Read();
    
    // 数据自动被 libvterm 解析并更新单元格
    
    // 标记窗口需要重绘
    Invalidate();
}
```

---

## 5. 通用实施步骤

以下步骤适用于**任何希望实现 AI 代理集成的经营管理游戏**。

### 5.1 前期准备与评估

#### 步骤 1：评估游戏架构

**关键问题**:

- [ ] 游戏是开源还是闭源？
- [ ] 使用什么语言/引擎？(C++, C#/Unity, C++/Unreal, Python, etc.)
- [ ] 是否有插件系统？
- [ ] 是否有现有的 API 或脚本接口？
- [ ] 游戏状态管理是否集中？
- [ ] 是否支持无头（headless）模式？

**决策矩阵**:

| 场景 | 推荐方案 | 难度 |
|------|---------|------|
| 开源 + 无插件系统 | Fork + 深度集成 | 中 |
| 开源 + 有插件系统 | 插件方式 | 低 |
| 闭源 + 有官方API | 使用官方API | 低-中 |
| 闭源 + 无API | 内存读取/注入 | 高 |

#### 步骤 2：定义代理能力边界

**设计原则**:

✅ **应该包含的能力**:
- 查询公开的游戏状态
- 执行玩家可以执行的操作
- 读取游戏内通知和提示
- 管理游戏内资源

❌ **不应包含的能力**:
- 直接修改内存或变量
- 绕过游戏规则和验证
- 访问隐藏信息（如迷雾下的地图）
- 使用调试或作弊命令（除非玩家也能用）

**能力分级示例**（以RTS游戏为例）:

| 级别 | 能力 | 说明 |
|------|------|------|
| L1 基础 | 查询资源、单位、建筑 | 只读信息 |
| L2 操作 | 建造、训练、移动单位 | 基本玩法 |
| L3 高级 | 研究科技、外交、编队 | 复杂策略 |
| L4 元游戏 | 保存/加载、设置 | 游戏管理 |

### 5.2 实施核心组件

#### 组件 A：通信层（必需）

**方案 1：JSON-RPC（强烈推荐）**

优点：
- 标准协议，LLM 容易理解
- 文本格式，易于调试
- 无状态，简单可靠

实现要点：
```cpp
// 伪代码示例
class GameRpcServer {
public:
    bool Start(int port = 9876) {
        _listener = CreateTcpSocket();
        _listener->Bind("127.0.0.1", port);
        _listener->Listen();
        return true;
    }
    
    void Update() {  // 在游戏主循环中调用
        // 接受新连接（非阻塞）
        AcceptNewClients();
        
        // 处理现有客户端的请求
        for (auto& client : _clients) {
            ProcessClient(client);
        }
    }
    
    void RegisterHandler(string method, HandlerFunc func) {
        _handlers[method] = func;
    }
    
private:
    TcpListener _listener;
    vector<Client> _clients;
    map<string, HandlerFunc> _handlers;
};
```

**方案 2：命名管道/共享内存**

适用场景：
- 需要超低延迟
- 仅限本地通信
- 平台特定实现可接受

**方案 3：REST API (HTTP)**

适用场景：
- 需要远程访问
- 语言互操作性重要
- 可以接受HTTP开销

#### 组件 B：CLI 工具（强烈推荐）

**为什么 CLI 重要**：
- AI 代理（尤其是 Claude Code）擅长使用 CLI
- CLI 易于测试和调试
- 可用于自动化脚本
- 人类也可以使用

**CLI 设计模式**：
```bash
# 推荐：名词-动词风格（类 kubectl）
<tool> <resource> <verb> [args] [flags]

# 示例
gamectl units list
gamectl building construct --type barracks --x 10 --y 20
gamectl resource get gold
gamectl research start gunpowder

# 或者：动词-名词风格（类 git）
<tool> <verb> <resource> [args] [flags]

# 示例
gamectl list units
gamectl construct building --type barracks
gamectl get resource gold
```

**CLI 实现架构**：
```
cli-tool/
├── src/
│   ├── main.cpp           # 入口，参数解析
│   ├── commands/
│   │   ├── registry.cpp   # 命令注册表
│   │   ├── units.cpp      # 单位命令
│   │   ├── buildings.cpp  # 建筑命令
│   │   ├── resources.cpp  # 资源命令
│   │   └── ...
│   ├── rpc/
│   │   └── client.cpp     # RPC 客户端
│   └── renderers/
│       ├── table.cpp      # ASCII 表格渲染
│       └── json.cpp       # JSON 输出
├── CMakeLists.txt         # 或 Makefile
└── README.md
```

**关键特性**：
- `--help` 无处不在
- `-o json` 支持机器可读输出
- 清晰的错误消息
- 一致的命名约定

#### 组件 C：终端集成（可选但推荐）

**方案 1：游戏内嵌终端（如本项目）**

优点：
- 无缝用户体验
- 可以同时看到游戏和AI操作
- 沉浸感强

需要：
- libvterm（终端仿真）
- PTY 支持（macOS/Linux: forkpty, Windows: ConPTY）
- 游戏渲染集成

实现复杂度：**高**

**方案 2：外部终端**

优点：
- 实现简单
- 平台兼容性好
- 可以使用标准终端

实现：
- AI 在独立终端窗口中运行
- 使用 CLI 工具与游戏通信

实现复杂度：**低**

推荐：**先从方案2开始**，稳定后再考虑方案1

#### 组件 D：状态暴露层

**设计原则**：

1. **分领域组织**
   - game.*（游戏全局）
   - units.*（单位）
   - buildings.*（建筑）
   - resources.*（资源）
   - map.*（地图）
   - ...

2. **粗粒度 API**
   - ❌ 不好：getUnitX(), getUnitY(), getUnitHealth() （3次调用）
   - ✅ 好：getUnit(id) → {x, y, health, ...} （1次调用）

3. **支持过滤和排序**
   ```json
   {
     "method": "units.list",
     "params": {
       "filter": {"type": "worker", "state": "idle"},
       "sort": "health",
       "order": "asc",
       "limit": 10
     }
   }
   ```

**API 示例**（以 4X 策略游戏为例）：

```jsonc
// game.status - 游戏全局状态
{
  "turn": 145,
  "gameSpeed": "normal",
  "paused": false,
  "era": "medieval",
  "victory": {
    "type": "domination",
    "progress": 0.35
  }
}

// resources.list - 资源列表
{
  "resources": [
    {
      "type": "gold",
      "amount": 1000,
      "income": 50,
      "upkeep": 30
    },
    {
      "type": "food",
      "amount": 500,
      "production": 20,
      "consumption": 15
    }
  ]
}

// units.list - 单位列表（支持分页和过滤）
{
  "units": [
    {
      "id": "unit_001",
      "type": "warrior",
      "position": {"x": 10, "y": 20},
      "health": 85,
      "movement": 2,
      "state": "idle"
    },
    // ...
  ],
  "total": 45,
  "page": 1,
  "hasMore": true
}

// map.tile - 查询地图格子
{
  "x": 10,
  "y": 20,
  "terrain": "grass",
  "elevation": 1,
  "visible": true,
  "explored": true,
  "improvement": "farm",
  "units": ["unit_001"],
  "resources": ["wheat"]
}
```

#### 组件 E：操作执行层

**设计原则**：

1. **使用游戏现有的 Action 系统**（如果有）
   - 确保验证逻辑一致
   - 支持多人游戏
   - 可以撤销/回放

2. **详细的错误信息**
   ```jsonc
   {
     "error": {
       "code": -32001,
       "message": "Cannot build: insufficient resources",
       "data": {
         "required": {"gold": 100, "wood": 50},
         "available": {"gold": 30, "wood": 50},
         "missing": {"gold": 70}
       }
     }
   }
   ```

3. **异步操作支持**（如果需要）
   ```jsonc
   // 请求
   {"method": "building.construct", "params": {...}}
   
   // 响应
   {
     "result": {
       "actionId": "action_123",
       "status": "inProgress",
       "estimatedCompletion": 5  // 回合数
     }
   }
   
   // 后续查询
   {"method": "action.status", "params": {"id": "action_123"}}
   ```



---

## 补充章节：实现细节深度解析

根据用户反馈，本章节详细解释以下关键问题：
1. rctctl CLI 工具具体是如何实现的
2. Claude Code 是如何集成的  
3. Claude Code 如何能调用 CLI（MCP、Skills 还是提示词工程）

### 一、rctctl CLI 工具的实现

#### 1.1 独立可执行程序，不是 RPC Handler

**关键结论**：`rctctl` 是一个**完全独立的可执行程序**，不是游戏内的 RPC handler。

这个设计决策带来的好处：

| 独立可执行程序 | RPC Handler（备选） |
|---------------|-------------------|
| ✅ 可在游戏外使用（测试、脚本） | ❌ 必须游戏运行 |
| ✅ AI直接shell调用 | ❌ 需实现RPC客户端 |
| ✅ 人类可读输出 | ❌ 只有JSON |
| ✅ 标准CLI模式 | ❌ 需自己实现 |
| ✅ 易于调试 | ❌ 需构造JSON |

#### 1.2 rctctl 的工作流程

```
1. 命令执行: rctctl park status
2. Python 进程启动
3. 解析参数: resource="park", action="status"
4. 构建 RPC 请求: {"method": "park.status", "params": {}}
5. 连接 TCP: localhost:9876
6. 发送请求 + 接收响应
7. 格式化输出（表格或JSON）
8. 进程退出
```

### 二、Claude Code 的集成机制

#### 2.1 集成方式：提示词工程 + PATH

**核心结论**：Claude Code **没有使用** MCP 或 Skills API。

**实际方式**：
1. ✅ **提示词工程** - 通过 `IN_GAME_AGENT.md` 告诉 Claude 有 rctctl 工具
2. ✅ **PATH 环境变量** - 让 `rctctl` 在 PATH 中可直接调用
3. ✅ **工作目录** - 设置为 `ai-agent-workspace/`
4. ✅ **标准 shell** - Claude 使用 `bash` 工具执行命令

#### 2.2 为什么不用 MCP？

| 提示词 + PATH | MCP |
|--------------|-----|
| ✅ 简单直接 | ❌ 需实现MCP服务器 |
| ✅ 所有AI适用 | ⚠️ 仅Claude |
| ✅ 人类也能用 | ❌ 专为AI设计 |
| ✅ 易于调试 | ❌ 需MCP客户端 |
| ✅ 稳定 | ⚠️ 协议可能变化 |

### 三、完整数据流示例

**场景：Claude 查询公园现金**

```
1. 用户问："公园有多少钱？"

2. Claude 内部推理
   - 系统提示词说有 rctctl 工具
   - 可用 rctctl park status 查询
   
3. Claude 执行
   bash(command="rctctl park status")
   
4. Shell 执行 rctctl
   - PATH 中找到 rctctl
   - 执行程序
   
5. rctctl 程序
   - 连接 localhost:9876
   - 发送 {"method": "park.status", ...}
   
6. 游戏内 RPC 服务器
   - 接收请求
   - 调用 ParkHandlers.status()
   - 访问 gameState.park
   - 返回 JSON
   
7. rctctl 格式化输出
   Park: My Park
   Cash: $12,500.00
   Rating: 678
   
8. Claude 读取并回复
   "公园有 $12,500 现金"
```

### 四、关键技术细节总结

1. **rctctl 是独立可执行文件** - 不是库、不是RPC handler
2. **通过 TCP socket 通信** - localhost:9876，JSON-RPC 2.0
3. **Claude 通过 PATH 调用** - 环境变量配置，无需MCP
4. **提示词作为API文档** - 教AI如何使用工具
5. **PTY + libvterm** - 提供真实终端环境
6. **符号链接到工作区** - 确保CLI在PATH中

这种设计的精妙之处：
- **对AI透明** - Claude只需知道有个命令行工具
- **对人类友好** - 可以手动运行相同命令
- **易于测试** - 不需要启动AI就能测试CLI
- **架构清晰** - 游戏、RPC、CLI、AI各司其职

---

本补充章节详细回答了用户的所有问题。
