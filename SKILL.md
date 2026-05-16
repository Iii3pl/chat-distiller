---
name: chat-distiller
description: 从微信/钉钉/飞书群聊和私聊中蒸馏人物画像、生成每日摘要。支持增量读取、6层人格分析、一键导出 Review Skill。
argument-hint: "[目标人物名 / 群组名 / --daily-digest / --dm 人名]"
version: "2.1.0"
user-invocable: true
---

# Chat Distiller — 聊天记录蒸馏器

> ⚠️ **Sandbox 注意**：wx-cli 需要访问 `~/Library/Containers/com.tencent.xinWeChat/` 和 `~/.wx-cli/`，这些路径在 Claude Code 默认沙箱外。所有 `wx` 命令需要 `dangerouslyDisableSandbox: true`。dingwave 同理需要访问 `~/Library/Application Support/DingTalkMac/`。

从微信、钉钉、飞书的群聊和私聊中提取人物特征，生成结构化摘要，追加式合并到 6 层人格画像。

## Execution Protocol（执行协议）

每次调用遵循以下顺序，**不要跳过**：

1. **Preflight** — 运行 Hook 1-3 的环境检查。三个平台全部检查，不跳过。
2. **Report** — 向用户展示检查结果：
   ```
   [✓] wx-cli      v0.1.9 已初始化
   [✗] dingwave    未找到
   [✓] lark-cli     已认证
   ```
3. **Block** — 如果请求的模式依赖某个未通过的平台，停止并引导用户修复。
4. **Resolve** — 对每个失败项，按对应 Hook 的恢复步骤引导用户。**绝对不要自己执行 `sudo` 命令**。
5. **Proceed** — 所有必需平台通过后，进入用户请求的工作模式。

## Progress Reporting

每个操作步骤向用户报告进度：
- "正在检查环境..."
- "正在拉取微信群消息（上次运行后新增 50 条）..."
- "正在拉取钉钉群消息（12 条新消息）..."
- "正在生成结构化摘要..."
- "保存到 data/2026-05-16/digest.md"

单步超过 30 秒时发送中间状态（"已处理 200/500 条..."）。

## 首次使用：环境检查

### Hook 1：微信环境检查

**CLI 查找顺序**：先查 repo 自带的 `./bin/wx`，再查 `./node_modules/.bin/wx`，最后用系统 PATH 的 `wx`。

```bash
# 查找 wx-cli
WX=$(command -v ./bin/wx || command -v ./node_modules/.bin/wx || command -v wx || echo "")
if [ -z "$WX" ]; then echo "NOT_INSTALLED"; else $WX --version; fi

# 检查初始化
$WX sessions 2>&1 && echo "INIT" || echo "NOT_INIT"
```

根据结果向用户展示：
- **已安装+已初始化** → 绿色 ✓，显示版本号
- **已安装+未初始化** → 引导用户完成以下 4 步（**不要自己执行 sudo**）：

> 以下步骤需要你在终端中手动完成：
> 1. `sudo codesign --force --deep --sign - /Applications/WeChat.app`
> 2. `killall WeChat && open /Applications/WeChat.app`
> 3. `sudo wx init`
> 4. `sudo chown -R $(whoami) ~/.wx-cli`
>
> 完成后告诉我，我重新检查。

- **未安装** → 引导安装：`npm install -g @jackwener/wx-cli`

### Hook 2：钉钉环境检查

> ℹ️ 钉钉聊天记录读取使用 [dingwave](https://github.com/Iii3pl/dingwave)。如无法获取，可跳过钉钉功能，仅使用微信+飞书。

```bash
DW=$(command -v ./bin/dingwave-cli || command -v ./node_modules/.bin/dingwave-cli || command -v dw || echo "")
if [ -z "$DW" ]; then echo "NOT_FOUND"; else ls ~/.dingwave/decrypted/*_dingtalk_decrypted.db 2>/dev/null || echo "NO_DB"; fi
```

根据结果引导：
- **已就绪** → 绿色 ✓
- **未解密** → 引导：1) 下载 dingwave 2) `dw doctor --json` 3) `dw decrypt`
- **未找到工具** → 提供 GitHub 链接，允许跳过

### Hook 3：飞书环境检查

```bash
LARK=$(command -v ./bin/lark-cli || command -v ./node_modules/.bin/lark-cli || command -v lark-cli || echo "")
if [ -z "$LARK" ]; then echo "NOT_INSTALLED"; else $LARK auth status 2>&1; fi
```

- **未安装** → `npx @larksuite/cli@latest install`
- **未认证** → `lark-cli auth login --recommend`

## 工作模式

### 模式 0：私聊 / 1对1 消息

```
/chat-distiller --dm 王小明
```

拉取与指定联系人的微信+钉钉私聊记录。私聊对人物蒸馏尤其有价值——群聊展现协作模式，私聊展现真实性格。
- **微信私聊**：`wx history "联系人备注" --chat-type private -n 200 --json`
- **钉钉私聊**：`dw history "{uid}" -n 200 --json`（需先 `dw resolve user "姓名"` 获取 uid）

### 模式 1：每日摘要（Daily Digest）

```
/chat-distiller --daily-digest
```

流程：
1. 从群聊列表（EXTEND.md 的 `group_list` 或自动发现）读取活跃群
2. 增量拉取微信 + 钉钉新消息（自 `history.json` 记录的 `last_message_time` 之后）
3. 生成结构化摘要（按项目/客户分组）
4. 保存到 `{data_root}/{date}/digest.md`
5. 更新 `history.json`

> 如配置了飞书多维表格，可从群聊映射表中自动发现群组。未配置时使用 EXTEND.md 的 `group_list` 手动指定。

### 模式 2：人物蒸馏（Persona Distill）

```
/chat-distiller 王小明
/chat-distiller --group "项目Alpha讨论群"
```

流程：
1. 定位目标人物所在的群（从 EXTEND.md 的 `group_list` 或飞书映射表）+ 私聊
2. 拉取微信 + 钉钉消息
3. 按 6 层框架提取 delta（L0-L5）
4. **追加式合并**到已有 persona.md。如无 persona.md → 自动调用 `/add-profile` 创建骨架再蒸馏
5. 标注 `[日期] [信源: 平台/群名] [置信度]`

### 模式 3：批量回溯（Backfill）

```
/chat-distiller --backfill --group "项目Alpha讨论群"
```

拉取指定群的全量历史消息，批量生成摘要和画像。

### 模式 4：导出 Review Skill

```
/chat-distiller --export-skill <人名>
```

将已蒸馏的人物画像导出为独立的审稿 SKILL.md，可直接发布到 GitHub。导出包含：6 层画像摘要 + 审稿规则 + 送审话术模板。

## 配置

### 交互式配置（无 EXTEND.md 时）

如果没有 EXTEND.md，引导用户逐项填写。每项先尝试自动检测，检测不到再提问：

1. **self_wxid / self_display** — 提问
2. **wx_cli_path** — `which wx` 自动检测
3. **dw_path / dw_db** — 检查 `~/bin/`、`~/.dingwave/decrypted/*.db`
4. **lark_base_token / lark_table_id** — 提问（需先在飞书创建多维表格）
5. **data_root** — 默认 `./data`
6. **default_since** — 默认当天往前 7 天

收集完毕后创建 EXTEND.md。

### EXTEND.md 示例

```yaml
# 微信
wx_cli_path: /opt/homebrew/bin/wx
self_wxid: wxid_xxxxxxxxxxxxx
self_display: 你的微信昵称

# 钉钉
dw_path: /path/to/dingwave-cli
dw_db: /Users/xxx/.dingwave/decrypted/xxx_dingtalk_decrypted.db

# 飞书（可选——如不使用飞书多维表格，删掉此段）
lark_base_token: your_lark_base_token_here
lark_table_id: your_lark_table_id_here

# 存储
data_root: ./data
default_since: 2026-06-01

# 群聊列表（如不使用飞书自动发现，在此手动指定）
group_list:
  - group_name: "项目Alpha讨论群"
    platform: wechat
    group_id: "123456@chatroom"
  - group_name: "客户Beta对接群"
    platform: dingtalk
    group_id: "abc123def456"
```

## 增量读取机制

每个群维护 `{data_root}/{group_id}/history.json`：

```json
{
  "group_id": "123456@chatroom",
  "group_name": "项目Alpha讨论群",
  "last_message_time": "2026-06-01T15:30:00",
  "last_digest_file": "2026-06-01.md",
  "message_count": 500
}
```

## 6 层 Persona 蒸馏框架

| Layer | 提取内容 | 信源优先级 |
|-------|----------|-----------|
| L0 核心性格 | 反复出现的态度、价值观 | decisions > casual |
| L1 身份 | 角色变化、职责调整 | decisions |
| L2 表达风格 | 语气、常用语、沟通节奏 | casual |
| L3 行为模式 | 推进方式、处理问题手法 | decisions > casual |
| L4 决策倾向 | 优先级排序、取舍逻辑 | decisions > long_form |
| L5 人际互动 | 对上下级/客户的不同模式 | casual |

置信度标尺：`high`（≥3 个独立信源）| `medium`（1-2 个）| `low`（单次观察）

## 输出示例

### 每日摘要

```markdown
# 消息摘要 2026-06-01

📋 TL;DR: 项目Alpha视频过审，客户Beta新选题推进

## 项目Alpha讨论群（微信）
- 刘运营：今日发布3条
- 赵策略：后续希望加强情绪向内容

## 客户Beta对接群（钉钉）
- 陈工：物料已收到，可以安排拍摄
```

### 人物蒸馏 Delta

```markdown
- [2026-06-01] [信源: 微信/项目Alpha群] [置信度: medium] 
  赵策略在选题确认时偏好看到策略说明，不只是看内容本身。
```

## 技能协作

发现新人时自动调用 `/add-profile` 创建骨架。蒸馏完成后自动调用 `/update-index` 重建索引。
如需创建项目页，运行 `/add-project`。

## 依赖

| 工具 | 安装 | 用途 |
|------|------|------|
| **wx-cli** | `npm install -g @jackwener/wx-cli` | 微信 Mac 版本地数据库读取 |
| **dingwave** | [github.com/Iii3pl/dingwave](https://github.com/Iii3pl/dingwave) | 钉钉 Mac 版本地数据库解密+读取 |
| **lark-cli** | `npx @larksuite/cli@latest install` | 飞书多维表格 + 文档读取 |


## 从蒸馏到 Review Skill

1. **选目标** — 确定要模拟的客户/同事
2. **蒸馏** — `/chat-distiller <人名>` 提取完整画像
3. **导出** — `/chat-distiller --export-skill <人名>` 生成标准格式
4. **发布** — 推到 GitHub，像 [zic-reviewer-skill](https://github.com/Iii3pl/zic-reviewer-skill) 一样

> 公开版本的 Review Skill 请使用化名，不要暴露真实姓名和群聊名称。
