# 🦞 wechat-assistant

**OpenClaw Skill** — 微信 AI 个人助手，自动从微信聊天中提取待办、日程、干货，推送到 Discord。

> ⚠️ **Alpha 半成品** — 核心链路已跑通，但未经大规模验证。
> 欢迎龙虾们在使用过程中提 Issue、提 PR，一起把这个东西做完善。

---

## 它能干什么

装完 Skill 后，跟你的 Agent 说 `帮我设置微信助手`，它会引导你完成配置。之后自动定时运行：

| 功能 | 频率 | 做什么 |
|------|------|--------|
| 📋 **待办提取** | 每 30 分钟 | 扫描私聊 → AI 识别待办 → 推送 Discord |
| 📅 **日程扫描** | 每 30 分钟 (8-23点) | 扫描私聊+工作群 → AI 识别日程 → 创建日历事件 |
| 📰 **干货收集** | 每天 9:00 | 扫描指定群聊 → AI 提炼干货 → 日报归档 |

**数据全程本地处理，不上传任何第三方服务。**

---

## 安装

```bash
openclaw skills install github:laolin5564/wechat-assistant
```

然后跟 Agent 说：

> 帮我设置微信助手

Agent 会按 SKILL.md 中的 Setup 流程一步步引导你：编译工具 → 提取密钥 → 解密数据库 → 首次同步 → 创建 Discord 频道 → 注册定时任务。

### 环境要求

- **macOS 13+**（ARM64 或 Intel）
- **微信桌面版 4.0+** 正在运行
- **Python 3.8+** + `pip3 install pycryptodome zstandard pyyaml`
- **sudo 权限**（仅密钥提取需要）

---

## 工作原理

### 架构

```
┌─────────────────────────────────────────────┐
│  OpenClaw Agent                              │
│                                              │
│  cron 定时触发 → 读 prompt 模板              │
│  → 调 CLI 拿 JSON → AI 分析                 │
│  → 推送 Discord / 创建日历事件               │
└──────────────────┬──────────────────────────┘
                   │ exec: python3 scripts/...
┌──────────────────▼──────────────────────────┐
│  CLI 工具层（纯 Python，不调 AI API）         │
│                                              │
│  refresh_decrypt.py → collector.py → extract │
│  解密 → 同步 → 提取 JSON 到 stdout           │
└─────────────────────────────────────────────┘
```

**关键设计：CLI 只负责数据提取（输出 JSON），AI 分析由 Agent 通过 prompt 模板驱动。**

这意味着你可以：
- 换 prompt 模板改变 AI 的分析方式，不用改代码
- 写新的 extract 脚本扩展功能，prompt 模板跟着加就行
- 不同的 Agent 模型分析同样的数据，对比效果

### 数据流

```
微信进程 → 加密 DB + WAL
                │
    ① refresh_decrypt.py ── WAL mtime 检测 → 增量 patch（<1秒）
                │
    ② collector.py --sync ── 增量同步到 collector.db
                │
    ③ extract_*.py ── 提取 JSON 到 stdout
                │
    ④ Agent 读 JSON ── 按 prompt 模板分析 → 推送 Discord
```

每次 cron 触发，Agent 按 `prompts/*.md` 模板依次执行 ①②③④。

### prompt 模板机制

`prompts/` 目录下的 `.md` 文件是 cron agentTurn 的指令模板。Agent 注册 cron 时，把模板中的占位符替换为实际值：

| 占位符 | 含义 |
|--------|------|
| `{{config_path}}` | config.yaml 绝对路径 |
| `{{skill_dir}}` | skill 根目录路径 |
| `{{thread_id}}` | Discord 子区 ID |
| `{{groups}}` | 监控群 ID（逗号分隔） |
| `{{ssh_host}}` | 微信所在机器的 SSH 地址（本机留空） |
| `{{ssh_password}}` | SSH 密码 |
| `{{obsidian_vault}}` | Obsidian vault 路径（可选） |

**想改 AI 分析逻辑？直接改 prompt 模板，不用动代码。**

---

## 增量解密：为什么不用每次全量解密 19GB

微信 4.0 用 SQLCipher 4 加密本地数据库（AES-256-CBC + HMAC-SHA512），总量约 19GB。

全量解密一次要 2-5 分钟，每 30 分钟跑一次不现实。

核心优化来自 [bbingz/wechat-decrypt](https://github.com/bbingz/wechat-decrypt/tree/feat/macos-support) 的 WAL 增量解密：

1. SQLite WAL 模式下，新消息写入 `.db-wal` 文件（预分配 4MB）
2. `refresh_decrypt.py` 检测 WAL 的 **mtime** 变化（不能用 size，因为是预分配的）
3. 只解密 WAL 中的新 frame（通过 salt 值校验有效性）→ patch 到已解密的 DB
4. 一个 4MB WAL 解密+patch 约 **70ms**

所以定时任务跑起来几乎无感知。

### 密钥过期处理

微信每次重启，进程内存中的密钥就变了。`refresh_decrypt.py` 在全量解密前会做 **HMAC 验证**：
- 验证通过 → 正常解密
- 验证失败 → exit code 2 + 告警，Agent 自动提醒你重新提取密钥

---

## 文件结构

```
scripts/
  decrypt/
    find_all_keys_macos.c    — 从微信进程内存提取加密密钥（C，需 sudo）
    decrypt_db.py             — 全量解密（首次设置用）
    config.py                 — YAML 配置加载器
  refresh_decrypt.py          — 增量解密（定时任务用，WAL patch）
  collector.py                — 消息增量同步到 collector.db
  extract_todos.py            — 私聊待办 → JSON
  extract_calendar.py         — 日程对话 → JSON
  extract_digest.py           — 群聊干货 → JSON
  requirements.txt            — Python 依赖
prompts/
  todo-scan.md                — 待办扫描 cron prompt 模板
  calendar-scan.md            — 日程扫描 cron prompt 模板
  digest.md                   — 干货收集 cron prompt 模板
config.example.yaml           — 配置模板
SKILL.md                      — OpenClaw Skill 定义（Agent 读这个）
```

---

## 已知限制

### 🔴 阻塞性

- **微信重启后密钥失效** — 需要重新 `sudo find_all_keys_macos`，目前无自动化方案
- **macOS Only** — 密钥提取只支持 macOS（Windows 需要不同的内存扫描）
- **需要 sudo 或 Full Disk Access** — 读微信数据库目录需要权限

### 🟡 待验证

- **refresh_decrypt.py 未在真实环境跑过** — WAL patch 在 mock 数据上测试通过
- **首次同步大量历史消息的性能** — 几十万条消息时未测
- **不同微信版本的 DB 结构差异** — 列名可能不同（已处理 `create_time` vs `CreateTime`）

### 🟢 改进方向

- Windows 支持
- 更多 extract 脚本（群活跃度分析、关键词告警…）
- MCP Server 集成
- 密钥自动刷新

---

## 参与共建

最需要帮助的方向：

1. **在你的机器上跑一遍** — 报告问题（不同微信版本、macOS 版本）
2. **新的 prompt 模板** — 改变 AI 分析方式，提升提取质量
3. **新的 extract 脚本** — 你想从微信里提取什么？写个脚本输出 JSON 就行
4. **Windows 适配** — 密钥提取和路径处理
5. **Bug 修复** — 代码 review 发现的问题

提 Issue 或直接 PR，代码风格不强求，能跑就行。

---

## 致谢

核心解密逻辑基于 [bbingz/wechat-decrypt](https://github.com/bbingz/wechat-decrypt/tree/feat/macos-support)：
- 进程内存密钥提取（`find_all_keys_macos.c`）
- SQLCipher 4 页面解密算法
- WAL 增量解密和 salt 校验机制

---

## License

MIT
