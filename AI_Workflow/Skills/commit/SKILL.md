---
name: commit
description: 仅在用户通过 /commit 显式调用时使用，不要自动触发。从实际改动生成 commit 说明并执行提交。
---

# Commit

从 git 实际改动生成 Conventional Commits 格式的说明，用户确认后执行提交。

## 触发约束

- 仅在 `/commit` 显式调用时执行本流程
- 用户口头要求提交（未走 /commit）时，只按 CLAUDE.md 的确认闸门执行，
  不套用本流程，但 Message 硬性规则依然生效
- 禁止修完代码顺手提议提交

## 流程

1. **摸清现场**：
   - `git status` 确认改动范围；有未暂存改动时列出，问用户是否一并提交
   - `git diff --staged`（或 `git diff`）读实际改动，**禁止凭会话印象写说明**
2. **判断改动性质**：
   - 混合改动（如 feat 和 fix 搅在一起、无关文件混入）→ 建议拆分提交，等用户决定
   - 改动过大无法一句话概括 → 同样建议拆分
3. **起草说明**：
   - 格式：`type: 中文描述`
   - type 从 `feat / fix / refactor / docs / test / style / chore` 中选
   - 描述写"做了什么"，不写"为什么"（原因复杂时另起一行正文说明）
4. **用户确认后才执行** `git commit`，确认闸门不可跳过

## Commit Message 硬性规则

- **禁止任何 trailer 字段**：不加 `Co-Authored-By`、`Signed-off-by`、
  `Generated with` 之类的署名/来源标注，一行都不加
- 只写用户改动本身的信息
- 遵循 Conventional Commits，中文描述

## 禁止事项

- 禁止 `git commit --amend`、`git push`——用户明确要求时另行确认
- 禁止提交未读过的文件（`git add -A` 前必须知道里面有什么）
- 禁止编造改动内容：diff 里没出现的不许写进说明
