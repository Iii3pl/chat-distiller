# Chat Distiller

从微信、钉钉、飞书聊天记录中蒸馏人物画像、生成每日摘要。

## 独有优势

- **三平台**：微信（wx-cli）+ 钉钉（dingwave）+ 飞书（lark-cli）
- **飞书骨架**：通过飞书多维表格自动定位每个人物在哪些群活跃
- **6 层 Persona**：Layer 0-5 + 纠错层，追加式合并
- **增量读取**：每群维护 history.json，只拉新消息
- **首次引导**：内置环境检查 hook，引导安装 wx-cli / dingwave / lark-cli

## 依赖安装

```bash
# 微信
npm install -g @jackwener/wx-cli

# 钉钉（联系管理员获取 dingwave）
# 解密后数据库位于 ~/.dingwave/decrypted/

# 飞书
npm install -g @anthropic/lark-cli
lark-cli auth login --domain base
```

## 快速开始

### 1. 首次配置

调用任意命令会自动触发环境检查，引导你完成配置。或手动创建：

```bash
cp EXTEND.md.example EXTEND.md
```

### 2. 生成每日摘要

```
/chat-distiller --daily-digest
```

### 3. 蒸馏人物画像

```
/chat-distiller 林小靓
```

### 4. 批量回溯

```
/chat-distiller --backfill --group "京东官号视频对接群"
```

## 制作你自己的 Review Skill

1. **选目标** — 确定要模拟的客户/同事
2. **蒸馏** — `/chat-distiller {人名}` 提取完整 6 层画像
3. **定义审稿规则** — 从 L0/L4/雷区 提取审核标准
4. **发布** — 参考 [zic-reviewer-skill](https://github.com/Iii3pl/zic-reviewer-skill) 格式

### 通版模板

```markdown
---
name: {target}-reviewer
description: 模拟 {姓名} 的审稿视角
user-invocable: true
---

# {姓名} Reviewer

## 角色设定
[从 Chat Distiller 蒸馏的 L0-L1]

## 审稿总原则
[从 Layer 4 提取]

## 高敏感点
[从雷区+反馈模式提取]
```

### 已发布

- [zic-reviewer-skill](https://github.com/Iii3pl/zic-reviewer-skill) — 未来生活实验室 Zic 审稿

## 许可

MIT
