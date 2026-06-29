# generate-commit-message

分析 git 改动并生成中文 Conventional Commits 提交信息（含破坏性变更 A/B 选项）。

## 安装

```bash
pnpx skills add czq297297/generate-commit-message-skill@generate-commit-message -g -y --agent cursor
```

或使用完整 URL：

```bash
pnpx skills add https://github.com/czq297297/generate-commit-message-skill -g -y --agent cursor -s generate-commit-message
```

## 用法

在 Cursor 中说：

- `/generate-commit-message`
- 「生成提交注释」「写 commit message」

默认只生成注释，不执行 `git commit`；明确要求「提交」时才会 commit。
