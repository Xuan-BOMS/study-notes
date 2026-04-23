# Office-Word-MCP-Server 配置入门说明

## 1. 这次为什么会恢复正常

你最初的配置是：

```json
{
  "command": "uvx",
  "args": [
    "--from",
    "office-word-mcp-server",
    "word_mcp_server"
  ],
  "type": "stdio"
}
```

这个写法参考的是项目 README，语法本身没有问题，但在实际运行里有几个不稳定因素：

1. `uvx` 会临时创建运行环境，并解析、安装依赖。
2. `office-word-mcp-server` 当前会拉到较新的 `fastmcp 3.x`。
3. 这个项目本身更适合 `fastmcp 2.x` 生态。
4. 某些 MCP 客户端对启动阶段更严格，只要启动时输出异常日志、安装信息、banner，或者进程关闭过快，就会判定握手失败。

后来能正常，核心原因通常是下面几条共同生效：

1. 你把依赖固定到了 `fastmcp<3`，避免了与当前客户端或项目代码不完全兼容的 `3.x`。
2. `uvx` 在本机已经有缓存，后续启动更快，不容易在握手阶段超时。
3. 运行环境稳定后，服务能持续挂住，客户端就能完成 `initialize` 握手。

简化理解：

- 之前：配置格式对，但运行时环境不稳定。
- 现在：配置格式对，依赖版本也更稳，所以能完成握手。

## 2. MCP 启动时到底发生了什么

MCP 的 `stdio` 模式本质上是：

1. 客户端启动一个外部进程。
2. 客户端通过该进程的标准输入 `stdin` 发送 JSON-RPC 请求。
3. 服务端通过标准输出 `stdout` 返回 JSON-RPC 响应。
4. 双方先完成 `initialize` 握手。
5. 握手成功后，客户端再读取工具列表、资源列表等能力信息。

你看到的报错：

```text
handshaking with MCP server failed: connection closed: initialize response
```

它表示的不是“配置 JSON 语法错了”，而是：

1. 客户端已经把服务拉起来了。
2. 客户端发了 `initialize`。
3. 但服务端没有在预期里成功回一个合法响应，或者进程提前退出了。

常见原因：

1. 进程刚启动就崩溃。
2. 依赖版本不兼容。
3. 首次安装过慢，客户端等不及。
4. `stdout` 被额外文本污染。
5. 客户端实现和服务端实现对协议细节的兼容性不足。

## 3. 你现在这份配置每一项是什么意思

你当前有效思路对应的配置大致是：

```json
{
  "command": "uvx",
  "args": [
    "--from",
    "office-word-mcp-server",
    "--with",
    "fastmcp<3",
    "word_mcp_server"
  ],
  "type": "stdio"
}
```

逐项解释如下。

### `command`

```json
"command": "uvx"
```

表示客户端实际要启动的可执行程序是 `uvx`。

`uvx` 的作用：

1. 临时创建隔离环境。
2. 下载并安装需要的 Python 包。
3. 直接运行该包暴露出来的命令。

它适合“我不想自己手动建虚拟环境，只想快速跑一个 Python 工具”的场景。

### `args`

```json
"args": [
  "--from",
  "office-word-mcp-server",
  "--with",
  "fastmcp<3",
  "word_mcp_server"
]
```

这是传给 `uvx` 的参数列表。

#### `--from office-word-mcp-server`

含义：

- 告诉 `uvx`：要从这个 Python 包里提供命令。

也就是说：

- 不是让系统直接找一个本地叫 `word_mcp_server` 的程序；
- 而是先安装/解析 `office-word-mcp-server` 这个包，再从这个包里找它声明的命令入口。

#### `--with fastmcp<3`

含义：

- 在运行这个包时，额外强制安装一个依赖约束：`fastmcp<3`。

为什么要加它：

1. 这个项目的包依赖写得比较宽，只要求 `fastmcp>=2.8.1`。
2. 这样一来，当前最新的 `fastmcp 3.x` 也会被选中。
3. 但这个项目本身和某些客户端组合时，用 `3.x` 容易出问题。
4. 所以手动把它钉在 `2.x`，能提高稳定性。

这类写法很常见，叫“版本钉住”或“版本约束”。

#### `word_mcp_server`

含义：

- 这是最终要执行的命令名。

它不是你随便起的名字，而是这个 Python 包在 `entry_points` 里暴露出来的控制台命令。

可以理解成：

```text
office-word-mcp-server 这个包
  -> 提供了一个命令
  -> 命令名叫 word_mcp_server
  -> 运行后会启动 MCP 服务
```

### `type: "stdio"`

```json
"type": "stdio"
```

表示客户端和服务端的通信方式是标准输入输出。

这时不会开一个 HTTP 地址给客户端连，而是直接通过进程管道通信。

它的优点：

1. 配置简单。
2. 不占端口。
3. 很适合本地桌面端客户端。

它的缺点：

1. 对启动日志更敏感。
2. 客户端超时阈值通常较低。
3. 首次安装或环境不稳定时更容易失败。

## 4. 为什么 `fastmcp<3` 这么关键

这个项目的包元数据大致是：

```text
Requires-Dist: fastmcp>=2.8.1
```

这表示作者只写了“最低版本”，没有写“最高版本”。

于是包管理器会倾向于选择最新版本，也就是现在的 `fastmcp 3.x`。

但现实里常出现这种情况：

1. 作者当时在 `2.x` 上开发和测试。
2. 后来 `fastmcp 3.x` 发布了。
3. 这个项目没有继续维护或没有及时适配。
4. 结果就是“理论依赖满足，但实际行为不稳定”。

因此：

- `fastmcp>=2.8.1` 是“作者声明的最低门槛”；
- `fastmcp<3` 是“你为了稳定性补上的实际上限”。

## 5. 为什么第一次常常比第二次更容易失败

因为第一次运行 `uvx` 时，它要做的事很多：

1. 解析依赖。
2. 下载 wheel。
3. 创建缓存环境。
4. 安装包。
5. 再启动真正的 MCP 服务。

而客户端通常只看到一件事：

- “我启动了一个 MCP 服务，它怎么还没回握手？”

如果客户端超时较短，就会直接报错。

第二次通常会快很多，因为：

1. 包已经下载到缓存。
2. 虚拟环境已经建立或可以复用。
3. 启动成本明显下降。

所以实际排障时，“先在终端手动跑一遍进行预热”是很有效的办法。

## 6. 以后配置 MCP 时，你应该优先检查哪些逻辑

建议按这个顺序检查。

### 第一步：确认命令本身能在终端跑起来

先不要急着放进客户端。

手动运行：

```powershell
uvx --from office-word-mcp-server --with "fastmcp<3" word_mcp_server
```

如果它能持续挂住，通常说明：

1. 命令存在。
2. 依赖可安装。
3. 服务没有启动即崩溃。

如果终端里都跑不起来，客户端里更不可能正常。

### 第二步：确认是 `stdio` 还是 HTTP/SSE

很多 MCP 服务器支持多种传输方式。

最常见有：

1. `stdio`
2. `sse`
3. `streamable-http`

如果客户端要求的是本地命令型 MCP，通常配 `stdio`。
如果客户端要求填 URL，那通常要看服务端是否支持 HTTP/SSE。

不要混用。

### 第三步：检查依赖是否需要手动钉版本

尤其是 Python、Node、Rust 等生态里的 MCP 服务，经常出现：

1. README 写法能跑。
2. 但几个月后依赖已经升级。
3. 原仓库没人维护。
4. 结果今天照抄 README 反而启动失败。

这时要优先查：

1. 包依赖范围是不是很宽。
2. 仓库是不是归档了。
3. issue 里有没有提到新版本不兼容。

### 第四步：区分是配置错，还是运行期失败

如果是配置格式错，通常客户端会报：

1. JSON 解析失败。
2. 缺少字段。
3. command/args 类型错误。

如果是运行期失败，通常会报：

1. process exited
2. connection closed
3. initialize response
4. timed out

这两类问题排查路径完全不同。

## 7. 常见参数和概念入门

### `uvx`

一句话理解：

- 临时运行 Python 包里的命令。

你可以把它理解成：

```text
不用手动建 venv，不用手动 pip install 到当前项目
直接下载后运行
```

### `--from`

指定“命令来自哪个包”。

### `--with`

追加额外依赖或额外版本约束。

示例：

```powershell
uvx --from some-package --with "dep<2" some-command
```

意思是：

- 启动 `some-package` 提供的命令；
- 但同时强制把 `dep` 控制在 2 以下。

### 入口命令

例如这里的 `word_mcp_server`。

它不是包名，而是包暴露出来的命令行入口。

很多 Python 包都是这样：

1. 包名和命令名不完全一样。
2. 真正运行时靠 `entry_points` 映射。

### `stdio`

进程管道通信模式。

### 握手 `initialize`

MCP 客户端连接服务端后，第一步一般会发送初始化请求。
服务端要回应自己的协议版本、能力列表、服务器信息。

只有这一步成功，客户端才会认为“这个 MCP 服务可用”。

## 8. 后续配置时的注意事项

下面这些是最实用的经验。

### 1. 优先固定关键依赖版本

如果仓库不活跃，或者已经归档，优先考虑：

1. 固定主依赖版本。
2. 固定 Python 版本。
3. 固定运行命令。

不要长期依赖“永远拉最新版本还能正常工作”。

### 2. 首次安装建议手动预热

对 `uvx`、`npx`、`cargo install` 这类即时安装工具尤其重要。

先在终端跑一遍，再交给客户端。

### 3. 看懂“包名”和“命令名”不是一回事

这里：

1. 包名是 `office-word-mcp-server`
2. 命令名是 `word_mcp_server`

以后遇到类似项目，先查 README、`pyproject.toml`、`entry_points`。

### 4. `stdio` 模式怕启动噪声

有些客户端对下面这些东西敏感：

1. banner
2. 调试日志
3. 首次安装提示
4. warning

如果发现“终端手动能跑，但客户端总说握手失败”，要怀疑：

1. 启动太慢；
2. 有输出噪声；
3. 客户端实现比较严格。

### 5. 长期使用时，本地虚拟环境通常比 `uvx` 更稳

`uvx` 很适合快速试用。
但如果你准备长期使用某个 MCP 服务，更稳的做法通常是：

1. 手动建一个固定虚拟环境；
2. 只装你验证过的版本；
3. 客户端直接调用本地 `python` 或本地脚本。

优点：

1. 启动更快；
2. 不依赖每次动态解析；
3. 版本更可控；
4. 更容易排障。

### 6. 尽量记录“最后能用的版本组合”

建议给自己留一份可复用记录，例如：

```text
office-word-mcp-server==1.1.11
fastmcp<3
Python 3.11
transport=stdio
```

以后电脑重装、迁移客户端、换项目时，会省很多时间。

## 9. 推荐的排障流程

以后遇到 MCP 服务启动失败，可以直接按这个流程走。

### A. 终端单独运行

确认命令本身能启动。

### B. 观察是否首次安装过慢

如果首次失败，第二次成功，优先怀疑缓存和超时。

### C. 固定依赖版本

尤其当项目仓库较老或已归档。

### D. 看服务端支持什么传输方式

确认客户端要的是 `stdio` 还是 URL。

### E. 必要时换成固定本地环境

如果 `uvx` 不稳定，就自己建 venv。

## 10. 一个更稳的长期方案

如果你后面要长期使用这个 Word MCP，推荐保存成固定环境。

示意步骤：

```powershell
py -3.11 -m venv %USERPROFILE%\mcp-word
%USERPROFILE%\mcp-word\Scripts\pip install "office-word-mcp-server==1.1.11" "fastmcp<3"
```

然后让客户端直接调用这个环境中的命令或 Python。

这样比 `uvx` 的优点是：

1. 启动更可预测。
2. 不受远端最新包变化影响。
3. 换客户端时也更容易复用。

## 11. 你这次可以记住的结论

最重要的结论只有四条：

1. 你的原始配置格式没有写错。
2. 真正的问题是运行期依赖版本和客户端兼容性，而不是 JSON 结构。
3. `fastmcp<3` 是这次稳定运行的关键修正。
4. `uvx` 适合快速试用，但长期稳定使用建议固定本地环境和版本。

## 12. 你之后可直接参考的命令

### 临时测试

```powershell
uvx --from office-word-mcp-server --with "fastmcp<3" word_mcp_server
```

### 指定 Python 版本测试

```powershell
uvx --python 3.11 --from office-word-mcp-server --with "fastmcp<3" word_mcp_server
```

### 长期固定环境

```powershell
py -3.11 -m venv %USERPROFILE%\mcp-word
%USERPROFILE%\mcp-word\Scripts\pip install "office-word-mcp-server==1.1.11" "fastmcp<3"
```

## 13. 术语速查

### MCP

Model Context Protocol，客户端和工具服务之间的标准通信协议。

### MCP client

例如 Claude Desktop、cc switch 这类会去启动或连接 MCP 服务的程序。

### MCP server

提供工具、资源、提示能力的外部服务进程。

### stdio

通过进程标准输入输出通信。

### JSON-RPC

MCP 常用的消息格式，客户端和服务端都发 JSON 消息。

### handshake

握手阶段，通常就是先完成 `initialize`。

### dependency pinning

依赖版本钉住，例如 `fastmcp<3`。

---

如果你后面还要配置别的 MCP 服务，可以直接把这份文档当模板用：先确认命令能单跑，再确认传输方式，再固定关键依赖版本，最后再接入客户端。
