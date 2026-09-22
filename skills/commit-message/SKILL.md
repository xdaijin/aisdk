---
name: commit-message
description: 按 Conventional Commits 规范生成 Git 提交信息并执行提交。当用户要求生成/撰写提交信息、提交代码（"commit"、"提交一下"、"帮我提交"）时使用。
---

# Git 提交信息生成规范

按 Conventional Commits 规范生成提交信息并提交。

## 工作流程

1. 并行运行以下命令收集信息：
   - `git status`（确认是否有 git 仓库及待提交文件）
   - `git diff --staged`（查看已暂存变更；若为空且用户意图是提交全部改动，则看 `git diff`）
   - `git log -5 --oneline`（参考项目已有的提交风格：语言、scope 习惯）
2. 分析变更，判断改动属于**一个原子提交**还是多个逻辑变更：
   - 若明显是多件事（如 feat + fix + 无关重构），先告知用户建议拆分，询问后再操作
3. 生成提交信息，执行 `git add` + `git commit`（除非用户只要信息不要提交）
4. 提交后运行 `git log -1 --stat` 确认结果

## 提交信息格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Header（必填，≤72 字符）：**

| type | 使用场景 |
|---|---|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `perf` | 性能优化 |
| `refactor` | 重构（不改变外部行为） |
| `docs` | 文档 |
| `style` | 格式（空格、分号，不改逻辑） |
| `test` | 测试 |
| `build` | 构建系统、依赖（pom/gradle/package.json） |
| `ci` | CI/CD 配置 |
| `chore` | 其他杂项 |
| `revert` | 回滚 |

- `scope`：影响模块（如 `order`、`user`、`auth`），与项目历史习惯保持一致
- `subject`：祈使句，结尾不加句号；说明"做了什么"而非"怎么做"

**Body（简单改动可省）：** 空一行后写动机与前后对比，不复述 diff。

**Footer：**
- 关联 issue：`Closes #123`、`Refs #456`
- 破坏性变更：type 后加 `!`，且 footer 写 `BREAKING CHANGE: <说明>`

## 硬性规则

- 一个提交只做一件事，可独立 revert
- 语言与项目历史保持一致（历史是中文就用中文，英文就用英文）
- 禁止无意义信息：`update`、`fix bug`、`修改代码`、`提交`
- 禁止提交时附带无关改动（如顺手格式化他人代码）
- 提交信息末尾追加：

  ```
  Co-Authored-By: Claude <noreply@anthropic.com>
  ```

- 不要执行 `git push`，除非用户明确要求
- 若当前在默认分支（main/master）上，先询问用户是否新建分支
- 使用 heredoc 写多行提交信息，避免引号转义问题：

  ```bash
  git commit -m "$(cat <<'EOF'
  feat(order): 新增订单超时自动取消

  超过 30 分钟未支付自动关闭并回滚库存。

  Closes #235

  Co-Authored-By: Claude <noreply@anthropic.com>
  EOF
  )"
  ```

## 示例输出

```
feat(auth): 新增 Token 刷新接口

旧 Token 即将过期时允许凭 refresh token 换新，
避免用户频繁重新登录。

Closes #128

Co-Authored-By: Claude <noreply@anthropic.com>
```
