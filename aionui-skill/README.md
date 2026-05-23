# AionUI Track Changes Skill — 上游合并前的临时方案

> 本文件在 PR 合并后可删除。

## 背景

OfficeCLI v5.10 已完整支持 Track Changes 修订模式（`--prop trackChange=ins/del/format`），但 AionUI 内嵌的 `office-cli` SKILL.md 尚未包含修订章节。导致 Agent 不知道 CLI 支持修订，会回退到 python-docx 等方案产生错误标记。

上游 PR 合并后，AionUI 更新依赖即可自动获得。合并前可通过用户 Skill 临时补齐。

## 安装

```bash
# 1. 创建 skill 目录
mkdir -p ~/.aionui-config/skills/officecli-track-changes

# 2. 下载 SKILL.md
curl -fsSL https://raw.githubusercontent.com/NextDoorLaoHuang-HF/OfficeCLI/feat/schema-track-changes/aionui-skill/SKILL.md \
  -o ~/.aionui-config/skills/officecli-track-changes/SKILL.md
```

## 启用

1. 打开 AionUI → 设置 → Skills
2. 找到 `officecli-track-changes`，确保已启用
3. 新建会话即可使用

如果使用 Word 文档助手，需要在助手设置中启用该 skill。

## 验证

在 AionUI 中新建 Word 文档助手会话，发送：

> 用修订模式修改 test.docx：插入"新增条款"，删除"旧条款"

Agent 应直接使用 `officecli add ... --prop trackChange=ins` 和 `--prop trackChange=del`，不再尝试 python-docx。

## 卸载

上游合并后，删除即可：

```bash
rm -rf ~/.aionui-config/skills/officecli-track-changes
```

## 分发

把 `SKILL.md` 文件发给朋友，放入 `~/.aionui-config/skills/officecli-track-changes/` 目录即可。
