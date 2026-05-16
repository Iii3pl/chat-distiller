---
name: chat-distiller
description: 从微信/钉钉/飞书聊天记录中蒸馏人物画像、生成每日摘要。支持增量读取、飞书多维表格骨架定位、6层人格分析。输入目标人物名或群组名即可。
argument-hint: "[目标人物名 / 群组名 / --daily-digest]"
version: "2.0.0"
user-invocable: true
---

# Chat Distiller — 聊天记录蒸馏器

> ⚠️ **Sandbox 注意**：wx-cli 需要访问 `~/Library/Containers/com.tencent.xinWeChat/` 和 `~/.wx-cli/`，这些路径在 Claude Code 默认沙箱外。所有 `wx` 命令需要 `dangerouslyDisableSandbox: true`。dingwave 同理需要访问 `~/Library/Application Support/DingTalkMac/`。

从微信、钉钉、飞书聊天记录中提取人物特征，生成结构化摘要，追加式合并到 6 层人格画像。

## 首次使用：环境检查

首次调用时，会自动运行环境检查，引导你完成各平台的数据源配置。

### Hook 1：微信环境检查

```bash
# 检查 wx-cli 是否安装
wx --version 2>&1 || echo "NOT_INSTALLED"

# 检查是否已初始化
wx sessions 2>&1 || echo "NOT_INIT"
```

**如果未安装**：
```
npm install -g @jackwener/wx-cli
```

**如果未初始化**（需要微信正在运行）：
```bash
# macOS：先确保微信已登录
# 1. 重签名微信（只需一次，微信更新后需重做）
sudo codesign --force --deep --sign - /Applications/WeChat.app

# 2. 重启微信
killall WeChat && open /Applications/WeChat.app

# 3. 初始化（提取数据库密钥）
sudo wx init

# 4. 修复权限（如果 ~/.wx-cli 被 root 占用）
sudo chown -R $(whoami) ~/.wx-cli
```

### Hook 2：钉钉环境检查

```bash
# 检查 dingwave 是否可用
ls /Users/$(whoami)/.dingwave/decrypted/*_dingtalk_decrypted.db 2>/dev/null || echo "NOT_FOUND"
```

**如果未解密**：
1. 确保钉钉 Mac 版已安装并登录
2. 获取 dingwave CLI（macOS arm64 二进制，放入 `~/bin/` 或 `/usr/local/bin/`）
3. 运行 `dw doctor --json` 检查钉钉数据目录
4. 运行 `dw decrypt` 解密数据库（首次需要钉钉登录状态）
5. 解密后数据库位于 `~/.dingwave/decrypted/{uid}_dingtalk_decrypted.db`

**故障排查**：
- `dw: command not found` → dingwave 不在 PATH，检查安装路径
- `database is locked` → 钉钉正在运行，先退出钉钉再试
- `no decrypted db found` → 运行 `dw decrypt` 重新解密
- 无法获取 dingwave → 在 Slack/微信联系运维或使用 [github.com/jackwener/wx-cli](https://github.com/jackwener/wx-cli) 参考实现自行编译

### Hook 3：飞书环境检查

```bash
# 检查 lark-cli 是否已认证
lark-cli auth status 2>&1 || echo "NOT_AUTH"
```

**如果未认证**：
```bash
lark-cli auth login --domain base --domain wiki
# base: 读取多维表格（群聊映射表）
# wiki: 读取飞书文档（可选）
```

## 工作模式

### 模式 1：每日摘要（Daily Digest）

```
/chat-distiller --daily-digest
```

流程：
1. 从飞书群聊映射表读取高优群列表
2. 增量拉取微信 + 钉钉新消息（自上次运行后）
3. 生成结构化摘要（按项目分组、优先级标注）
4. 保存到 `{data_root}/{date}/digest.md`
5. 更新 `history.json`

### 模式 2：人物蒸馏（Persona Distill）

```
/chat-distiller 林小靓
/chat-distiller --group "元宝传播@小题"
```

流程：
1. 从飞书群聊映射表定位目标人物的所有活跃群
2. 拉取微信 + 钉钉消息
3. 按 6 层框架提取 delta：
   - L0 核心性格
   - L1 身份
   - L2 表达风格
   - L3 行为模式
   - L4 决策倾向
   - L5 人际互动模式
4. 追加式合并到已有 persona.md
5. 标注 `[日期] [信源] [置信度]`

### 模式 3：批量回溯（Backfill）

```
/chat-distiller --backfill --group "京东官号视频对接群"
```

拉取指定群的全量历史消息，批量生成摘要和人物画像。

### 模式 4：导出 Review Skill

```
/chat-distiller --export-skill Zic
```

将已蒸馏的人物画像导出为独立的审稿 SKILL.md，可直接发布到 GitHub。  
导出包含：6 层画像摘要 + 审稿规则 + OracleProto 校准 + 送审话术模板。

## 配置

创建 EXTEND.md 自定义配置：

```yaml
# 微信相关
wx_cli_path: /opt/homebrew/bin/wx
self_wxid: wxid_xxxxx
self_display: 吴亮

# 钉钉相关
dw_path: /path/to/dingwave-cli
dw_db: /Users/xxx/.dingwave/decrypted/xxx_dingtalk_decrypted.db

# 飞书相关
lark_base_token: FMy8bZknzaSFTWsnQqIcTd6vnNd
lark_table_id: tbl1mn5LHpdhJ8do

# 数据存储
data_root: ./data

# 默认时间范围
default_since: 2026-05-01
```

不创建 EXTEND.md 时，使用交互式引导配置。

## 增量读取机制

每个群维护一个 `history.json`：

```json
{
  "group_id": "xxx@chatroom",
  "group_name": "元宝传播@小题",
  "last_message_time": "2026-05-13T15:30:00",
  "last_digest_file": "2026-05-13.md",
  "message_count": 500
}
```

下次运行时自动从 `last_message_time` 开始拉取，只处理新消息。

## 飞书骨架定位

飞书的群聊映射表是群组 → 项目 → 客户 → 部门的唯一映射源。

```bash
lark-cli base +record-list \
  --base-token {token} \
  --table-id tbl1mn5LHpdhJ8do \
  --limit 500 --format json
```

从表中获取：群名 → 平台(微信/钉钉) → 关联项目 → 客户 L1/L2/L3 → 部门 → 优先级(🔴高/🟡中/🟢低)

## 6 层 Persona 蒸馏框架

| Layer | 提取内容 | 信源优先级 |
|-------|----------|-----------|
| L0 核心性格 | 反复出现的态度、价值观 | decisions > casual |
| L1 身份 | 角色变化、职责调整 | decisions |
| L2 表达风格 | 语气、常用语、沟通节奏 | casual |
| L3 行为模式 | 推进方式、处理问题手法 | decisions > casual |
| L4 决策倾向 | 优先级排序、取舍逻辑 | decisions > long_form |
| L5 人际互动 | 对上下级/客户的不同模式 | casual |

置信度标尺：
- `high` — ≥3 个独立信源验证
- `medium` — 1-2 个信源清晰出现
- `low` — 单次观察

## 输出示例

### 每日摘要

```markdown
# 消息摘要 2026-05-13

📋 TL;DR: 京东JOY视频过审，元宝新选题推进

## 🔴 高优
### 京东官号视频对接群
- 母亲节视频卡审已解决，投流文档已更新
- 来贺：气偶拿回来了，可以安排拍摄

### 元宝传播@小题  
- 西多多：今日发布3条，矩阵号日常+蒜苔游戏+仿妆
- 侯文韬已确认选题方向

## 🟡 中优
...
```

### 人物蒸馏 Delta

```markdown
- [2026-05-13] [信源: 微信/元宝传播] [置信度: medium] 
  侯文韬在选题确认时偏好看到「为什么选这个方向」的策略说明，
  不只是看内容本身。
```

## 依赖

- **wx-cli**：`npm install -g @jackwener/wx-cli`（微信 Mac 版 4.x 需运行中）
- **dingwave**：本地钉钉数据库解密工具（联系管理员获取）
- **lark-cli**：`npm install -g @anthropic/lark-cli` 然后 `lark-cli auth login --domain base`

## 从蒸馏到 Review Skill

蒸馏出人物画像后，可以制作针对性的审稿 Skill：

1. **选目标**：确定要模拟的客户/同事
2. **蒸馏**：`/chat-distiller {人名}` 提取完整画像
3. **定义审稿规则**：基于 L0-L5 画像，写出该人物的敏感点、决策偏好、表达风格
4. **发布**：将画像 + 审稿规则打包为独立 SKILL.md

详见 [README.md](./README.md) 中的「制作你自己的 Review Skill」教程。
