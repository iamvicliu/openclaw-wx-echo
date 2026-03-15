# 待办扫描 — Cron agentTurn Prompt

## 任务

从微信私聊中提取待办事项，有变化则推送到 Discord。

## 执行步骤

### 1. 刷新解密 + 同步消息

> 如果 `{{ssh_host}}` 非空，所有 python3 命令需要通过 SSH 执行：
> `sshpass -p '{{ssh_password}}' ssh -o StrictHostKeyChecking=no {{ssh_host}} "cd {{skill_dir}}/scripts && python3 ..."`
> 如果为空，直接本地执行。

```bash
cd {{skill_dir}}/scripts

# 增量解密（WAL patch，通常 <1 秒）
# 如果退出码=2，表示密钥过期（微信重启过），发告警后终止
python3 refresh_decrypt.py --config {{config_path}}

# 同步到 collector.db
python3 collector.py --config {{config_path}} --sync
```

> **如果 `refresh_decrypt.py` 输出包含 "HMAC 验证失败" 或退出码为 2：**
> 发 Discord 告警：`⚠️ 微信密钥已过期，需要重新提取。请运行 sudo find_all_keys_macos`
> 然后**终止本次任务**，不继续后续步骤。

### 2. 提取私聊数据

```bash
python3 extract_todos.py --config {{config_path}}
```

> 输出 JSON 到 stdout，包含 `conversations` 和 `existing_todos`。

### 3. 分析 JSON 输出

从 conversations 中识别待办事项。

#### 什么算待办
- 对方**请求我做的事**（明确的 action item）
- **我承诺要做的事**（"好的我去处理"、"我来搞"）
- 涉及**金钱、合同、法律**的事项（urgent=true）
- 有**明确 deadline** 的事项（urgent=true）

#### 什么不算待办
- 纯聊天、寒暄、问好
- 已经当场解决的问题
- 咨询性质的对话（我在回答别人问题）
- 广告、推销、群发消息
- 纯表情、图片消息

#### 去重规则
- 检查 `existing_todos` 中是否已存在相似待办（同一联系人 + 相似 summary）
- 已存在的不重复添加
- 检查是否有待办在对话中被解决（resolved）

### 4. 更新 todos.json

todos.json 路径：从 config.yaml 的 `state.todos_file` 读取（默认在 config.yaml 同级目录的 `./todos.json`）。

用 `exec` 工具执行 Python 脚本读写：

```bash
python3 -c "
import json, os
path = '{{config_path}}'.replace('config.yaml', '') + 'todos.json'
# ... 读取、更新、写回
"
```

或直接用 `read` + `write` 工具操作文件。

更新规则：
- 新增的待办：`status: "open"`，含 `contact`, `summary`, `urgent`, `created`
- 已解决的：`status: "done"`，加 `resolved_date`

### 5. 推送到 Discord Forum

**只在有变化时**（新增或完成）推送到论坛频道 `{{todo_forum_id}}`。

> **发帖方式**：使用 OpenClaw `message` 工具，`action=thread-create`，`target={{todo_forum_id}}`，
> `threadName=帖子标题`，`message=帖子内容`，`appliedTags=["tag名称"]`。
>
> ⚠️ 如果 `message(action=thread-create)` 报错，改用 `exec` 执行 curl：
> ```bash
> curl -s -X POST "https://discord.com/api/v10/channels/{{todo_forum_id}}/threads" \
>   -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
>   -H "Content-Type: application/json" \
>   -d '{"name":"帖子标题","applied_tags":["tag_id"],"message":{"content":"帖子内容"}}'
> ```
> Tag IDs 需要先用 `curl GET /channels/{{todo_forum_id}}` 查 `available_tags` 获取。

#### 5a. 去重检查

发帖前，先搜索论坛已有帖子，避免重复：

```
message(action="thread-list", target="{{todo_forum_id}}")
```

检查返回的帖子列表，按**联系人 + 关键词**匹配。如果已有相同待办的帖子，跳过创建。

#### 5b. 新增待办 → 每条一个帖子

对每条**新增**待办，创建一个论坛帖子：

- **帖子标题**：
  - 紧急：`[🔴紧急] 联系人 — 待办摘要`
  - 跟进：`[🟡跟进] 联系人 — 待办摘要`
- **帖子内容**：
  ```
  📋 **待办详情**

  **联系人**：XXX
  **摘要**：待办描述
  **紧急程度**：🔴紧急 / 🟡跟进
  **创建时间**：YYYY-MM-DD HH:MM

  💬 **来源对话摘录**
  > 相关对话内容...
  ```
- **appliedTags**：
  - 紧急 → `["🔴紧急"]`
  - 跟进 → `["🟡跟进"]`

#### 5c. 已完成待办 → 回复原帖并标记

对每条**已解决**的待办：

1. 在 5a 的帖子列表中查找对应的原帖（按联系人 + 关键词匹配）
2. 如果找到原帖，用 `message(action="thread-reply", threadId=原帖ID, message="✅ 已完成 — YYYY-MM-DD")` 回复
3. 如果找不到原帖，跳过（不单独创建完成帖子）

如果没有变化（无新增、无完成），不发任何消息。
