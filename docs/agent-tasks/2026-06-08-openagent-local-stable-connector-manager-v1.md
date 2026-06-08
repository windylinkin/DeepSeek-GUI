# 小K私人版 v1：实现 OpenAgent Local 长期连接管理器

## 目标

把小K私人版从“随机 quick tunnel 临时连接”升级为“稳定 OpenAgent Local Connector 管理器”。这一步不是做完整 AgentOS，也不是重写 DeepSeek-GUI，而是在现有 exe 桌面版里增加一个长期可用的连接管理页面/模块，让林小逊以后只配置一次固定地址，ChatGPT 云端就能长期派任务到本地。

## 背景

当前 OpenAgent Local OAuth connector 使用 trycloudflare quick tunnel，每次启动都会变 URL，导致 ChatGPT connector 绑定旧地址后无法继续使用。长期方案必须支持固定公网入口，例如：

- Cloudflare Named Tunnel + 自有域名
- ngrok reserved/static domain
- 用户手动填写的固定 MCP URL

## 项目信息

- 小K项目：`D:\ai\deepseek-gui-agentos-private`
- 分支：`private/agentos`
- OpenAgent 项目：`D:\ai\openagent`
- OpenAgent 统一任务入口：`D:\ai\openagent\scripts\agent-task.ps1`
- OpenAgent connector 启动脚本：`D:\ai\openagent\scripts\start-openagent-chatgpt-connector-oauth.ps1`
- OpenAgent connector 停止脚本：`D:\ai\openagent\scripts\stop-openagent-chatgpt-connector.ps1`

## 核心要求

在小K桌面版中新增“OpenAgent Local 连接管理器”，让小K可以：

1. 显示当前 connector 状态
2. 管理固定 MCP URL 配置
3. 显示 ChatGPT connector 所需配置项
4. 启动/停止本地 connector
5. 显示权限模式 read / write / run
6. 解释 quick tunnel 只是临时模式，不作为长期正式入口
7. 为以后 ChatGPT 云端直接 `oa_create_task` 派任务打基础

## 页面建议

标题：

> 小K · OpenAgent Local 连接管理器

页面至少包含：

### 1. 连接状态卡片

- 本地 connector：online / offline / unknown
- 本地地址：`http://127.0.0.1:8790`
- MCP 地址：用户配置的固定 URL + `/mcp`
- OAuth Auth URL：固定 URL + `/oauth/authorize`
- OAuth Token URL：固定 URL + `/oauth/token`
- 当前权限模式：read / read+write / read+write+run
- 当前模式：quick tunnel / fixed url / named tunnel / custom

### 2. 配置卡片

建议新增：

- `agentos.local.example.json`
- `agentos.local.json`

`agentos.local.json` 必须 gitignore，不提交真实私人配置。

配置字段建议：

```json
{
  "openagentRoot": "D:\\ai\\openagent",
  "localBaseUrl": "http://127.0.0.1:8790",
  "publicBaseUrl": "",
  "connectorMode": "custom-fixed-url",
  "clientId": "openagent-local",
  "scopes": ["openagent.read"],
  "allowWrite": false,
  "allowRun": false
}
```

### 3. ChatGPT 配置说明卡片

根据 `publicBaseUrl` 自动展示：

```text
Connector URL: <publicBaseUrl>/mcp
Auth URL: <publicBaseUrl>/oauth/authorize
Token URL: <publicBaseUrl>/oauth/token
Client ID: openagent-local
Client Secret: 留空
Token endpoint auth method: none
Default scopes: openagent.read
```

如果 `publicBaseUrl` 为空，显示：

> 尚未配置固定公网入口。请配置 Cloudflare Named Tunnel / ngrok static domain / 其他固定 HTTPS URL。

### 4. 操作按钮

- 检查状态
- 启动本地 connector
- 停止本地 connector
- 打开 OpenAgent 项目目录
- 复制 ChatGPT 配置
- 查看最近日志

危险操作必须提示：

- 停止 connector 会断开 ChatGPT 云端连接
- teardown 可能清空剪贴板或停止本地服务

## 安全 IPC / 命令调用层要求

如果项目是 Electron：

- main process 负责 `child_process`
- renderer 不得直接调用 `child_process`
- preload 只暴露白名单 API

允许的命令白名单：

- 启动：`D:\ai\openagent\scripts\start-openagent-chatgpt-connector-oauth.ps1`
- 停止：`D:\ai\openagent\scripts\stop-openagent-chatgpt-connector.ps1`
- 状态：访问 `http://127.0.0.1:8790/health` 或调用已有 status 脚本
- 任务脚本：`D:\ai\openagent\scripts\agent-task.ps1 -Task status / commit-check / handoff / verify / verify-oauth`

命令执行要求：

- cwd 固定为 `D:\ai\openagent`
- 捕获 stdout/stderr
- 脱敏输出：Bearer token、sk-*、token=、api_key=、trycloudflare URL、ngrok URL、OAuth code
- 不打印完整环境变量
- 失败时返回错误，不允许页面白屏

## 页面文案要求

页面文案使用中文，面向林小逊理解。

必须解释清楚：

- quick tunnel：临时测试用，每次 URL 会变
- fixed URL：长期正式用，ChatGPT 只需配置一次
- read：只能查看项目
- write：可以写任务，例如 `oa_create_task`
- run：可以远程运行命令，风险更高，默认关闭

## 验证要求

至少验证：

1. 小K能正常启动
2. 新页面/入口能打开
3. 检查状态按钮不会报错
4. 未配置 `publicBaseUrl` 时，页面显示“尚未配置固定公网入口”
5. 已配置 `publicBaseUrl` 时，能正确生成 ChatGPT connector 配置
6. 启动/停止按钮有安全提示
7. 不影响原有聊天功能
8. `git diff` 只包含本任务相关文件
9. `agentos.local.json` 不会被提交

## 文档要求

新增文档：

`docs/openagent-local-stable-connector-manager-v1.md`

内容包括：

- 为什么 quick tunnel 不适合长期使用
- 小K连接管理器做了什么
- 如何配置 fixed publicBaseUrl
- ChatGPT connector 应该怎么填
- read / write / run 权限区别
- 如何启动和停止 connector
- 如何验证连接
- 已知限制
- 下一步建议

## 约束

- 不要重写 DeepSeek-GUI
- 不要大改 UI 架构
- 不要破坏现有聊天功能
- 不要默认开启 write/run 权限
- 不要自动开启公网 tunnel 后不清理
- 不要把 token、key、cookie、tunnel URL 写进日志
- 不要提交 git commit，除非用户明确要求
- 不要运行 `git add -A`
- 不要把固定域名写死成某个真实域名
- 不要把 quick tunnel 当成正式长期方案

## 最终报告格式

```markdown
## 完成结果
- 修改文件：
- 新增页面/入口：
- 配置文件：
- IPC/命令调用方式：
- 验证结果：

## 使用方法
1. 打开小K
2. 进入 OpenAgent Local 连接管理器
3. 填入固定公网 URL
4. 复制 ChatGPT connector 配置
5. 检查连接状态

## 已知限制

## 下一步建议
```
