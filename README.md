# AntCode

基于 .NET 8 WinForms + WebView2 的 Windows 桌面智能体：借助网页版 ChatHub作为 LLM，在本地项目中新建、修改、删除文件和文件夹，帮你编写代码和文档。

## 运行环境
- Windows 10/11，已安装 Microsoft Edge WebView2 Runtime（Win11 自带）
- .NET 8 SDK、Visual Studio 2022 17.8+

```powershell
cd AntCode
dotnet restore
dotnet run
# 发布单文件 EXE
dotnet publish -c Release -r win-x64 --self-contained false -p:PublishSingleFile=true
```
> 发布后 `Assets` 文件夹需要与 `AntCode.exe` 放在同一目录（项目已设置自动复制）。

## 使用流程
1. **认证页**：WebView2 打开 `ChatUrl`，检测到输入框 `#m365-chat-editor-target-element` 后自动进入下一步（也可以手动点“我已登录，继续”）。登录 Cookie 保存在 `%LocalAppData%\AntCode\WebView2`，下次启动会自动通过。
2. **项目选择页**：最近项目（可搜索、右键移除）、新建项目（名称 + 位置，在该位置下创建同名文件夹）、打开已有文件夹。
3. **主界面**（VS Code 风格）：左侧资源管理器，中间对话，右侧 ChatHub 调试视图（F12 切换）。

## 工作原理
```
用户输入 ─► PromptBuilder 组合“用户问你：… + 项目结构 + 工具列表 + 回答规则”
        ─► 写入 ChatHub 输入框（paste 事件 → CDP Input.insertText → execCommand，三级兜底）
        ─► 点击发送按钮（找不到则用 CDP 发送回车）
        ─► 等待完整回答
        ─► 取最后一个 [data-testid="copilot-message-div"]，忽略 [data-testid="chatQuestion"]
        ─► 提取其中全部 [aria-label="Code editor"]，逐个解析 JSON，按顺序执行工具
        ─► 将执行结果回传给 LLM，循环，直到 LLM 返回纯文本 → 显示在中间对话框
```

### 如何确保拿到“完整回答”
1. 发送前记录回答数量和最后一条回答的哈希值；
2. 阶段 1：等待出现新回答（数量增加或内容变化），超时为 `FirstTokenTimeoutSeconds`；
3. 阶段 2：页面上没有“停止生成”按钮，并且回答文本连续 `StableMilliseconds` 毫秒不变，才算完成；
4. 代码块像工具 JSON 但还解析不了时，最多再等 4 个稳定周期；
5. 总超时为 `ResponseTimeoutSeconds`，到时使用已获取的内容并给出提示。

## 工具（API）列表
| 工具 | 参数 | 说明 |
|---|---|---|
| create_file | path, content, overwrite? | 新建文件 |
| write_file | path, content | 覆盖写入整个文件 |
| append_to_file | path, content | 追加到末尾 |
| read_file | path, startLine?, endLine? | 读取带行号的内容 |
| replace_lines | path, startLine, endLine, content | 替换第 N~M 行（含两端） |
| insert_lines | path, line, content | 在第 line 行之前插入（line=总行数+1 表示追加） |
| delete_lines | path, startLine, endLine | 删除第 N~M 行 |
| replace_text | path, oldText, newText, replaceAll? | 精确文本替换 |
| delete_file | path | 删除文件（可确认，自动备份） |
| move_file | path, newPath | 移动或重命名文件 |
| copy_file | path, newPath | 复制文件 |
| create_folder | path | 新建文件夹（支持多级） |
| delete_folder | path, recursive? | 删除文件夹（可确认，自动备份） |
| move_folder | path, newPath | 移动或重命名文件夹 |
| list_directory | path, recursive?, maxDepth? | 列出目录 |
| search_text | query, path?, filePattern?, isRegex? | 在文件中搜索 |
| get_file_info | path | 获取文件或文件夹信息 |

- 工具名、参数名支持常见别名（如 `create_new_file`、`fileName`）；
- 支持单个对象、对象数组、`{"tool_calls":[…]}`、OpenAI `function` 写法；
- 自动修复 JSON 字符串中未转义的换行；仍无法解析时把错误回传给 LLM 重新输出；
- 路径限制在项目目录内；修改、删除前备份到 `%LocalAppData%\AntCode\backups`；
- 修改文件时保留原编码（BOM）和换行符（CRLF/LF）。

## 配置文件 `%AppData%\AntCode\settings.json`
| 字段 | 说明 |
|---|---|
| ChatUrl | ChatHub 地址 |
| ShowDebugView | 是否显示右侧 ChatHub 视图（调试用 true，后期改 false） |
| AutoContinue | 工具执行后是否自动把结果回传给 LLM |
| ConfirmDestructive | 删除前是否弹窗确认 |
| MaxAgentSteps | 单个任务最多执行多少轮 |
| StableMilliseconds / PollIntervalMilliseconds | 判断回答完成的稳定时长 / 轮询间隔 |
| FirstTokenTimeoutSeconds / ResponseTimeoutSeconds | 超时设置 |
| Selectors | 页面 DOM 选择器，页面改版时只改这里，无需重新编译 |
| IgnoredFolders | 文件树和工具中忽略的目录 |

> 隐藏 ChatHub 时并不真正设为不可见，而是铺在对话面板下层，同时关闭 Chromium 后台节流，避免页面被挂起导致回答中断。
