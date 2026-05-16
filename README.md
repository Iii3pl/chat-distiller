# Chat Distiller

从微信、钉钉、飞书的群聊和私聊中蒸馏人物画像、生成每日摘要。

## 独有优势

- **三平台**：微信（wx-cli）+ 钉钉（dingwave）+ 飞书（lark-cli）
- **群聊+私聊**：支持群聊和 1 对 1 私聊两种数据源
- **6 层 Persona**：Layer 0-5 + 纠错层，追加式合并
- **增量读取**：每群维护 history.json，只拉新消息
- **首次引导**：内置环境检查 hook + 交互式配置向导
- **一键导出**：蒸馏完成后直接导出为可发布的 Review SKILL.md

## 依赖安装

```bash
# 微信
npm install -g @jackwener/wx-cli

# 钉钉
# 从 https://github.com/Iii3pl/dingwave 下载或自行编译

# 飞书
npx @larksuite/cli@latest install
lark-cli auth login --recommend
```

## 快速开始

首次运行自动触发环境检查和交互式配置：

```
/chat-distiller --daily-digest          # 生成今日摘要
/chat-distiller 王小明                   # 蒸馏某人的画像
/chat-distiller --dm 王小明             # 拉取私聊记录
/chat-distiller --backfill --group "群名"  # 批量回溯
/chat-distiller --export-skill 王小明    # 导出为 Review Skill
```

## 制作你自己的 Review Skill

1. **安装** — 配置微信/钉钉数据源
2. **蒸馏** — `/chat-distiller <人名>` 自动拉取聊天记录，提取 6 层画像
3. **导出** — `/chat-distiller --export-skill <人名>` 生成标准格式
4. **发布** — 推到 GitHub

> ⚠️ 公开版本的 Review Skill 请使用化名，不要暴露真实姓名和群聊名称。

### 已发布的 Review Skill

- 更多示例欢迎 PR 贡献

## 许可

MIT
