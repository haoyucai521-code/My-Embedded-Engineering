# 个人 Skill 库

自写的 Claude Code Skill 归档，作为经验沉淀与版本管理。

**运行位置**：`~/.claude/skills/`（Claude Code 实际加载处）

**本目录**：归档位置。仓库是唯一编辑处，修改后手动同步到运行位置：

```bash
cp AI_Workflow/Skills/<name>/SKILL.md ~/.claude/skills/<name>/SKILL.md
```

## 收录标准

只收自己写的、包含"踩过坑才知道"内容的 Skill。官方/第三方 Skill（如 writing-skills）不归档。

## Skill 清单

| Skill | 用途 | 设计意图 |
|---|---|---|
| bringup | 新外设/新驱动开发的配置检查单 | TODO |
| bug-fix-summary | Bug 修复后的总结与经验提炼（事实层/经验层） | TODO |
| commit | 从实际改动生成 Conventional Commits 提交 | TODO |
| debug | 嵌入式运行故障分层定位（未定位根因前禁止改代码） | TODO |
| impact-review | 以 diff 为锚点顺调用链辐射的影响面审查 | TODO |
| memstat | map 文件与内存预算分析（RAM/Flash/栈） | TODO |

## 演进记录

- 2026-08-25：首次归档，从 `~/.claude/skills/` 拷贝 6 个 Skill
