---
name: git-push
description: 'Git add, commit, push workflow for the current working directory. Use when the user asks to commit, save to git, push code, or submit to GitHub. Supports both user-specified commit messages and AI-generated conventional commit messages from diff analysis.'
disable-model-invocation: true
user-invocable: true
---

# Git Push

> ⚠️ **触发规则：此 skill 仅通过 `/git-push` 斜杠命令触发，绝不会自动调用。**
> 即使用户说「代码写完了」「帮我保存」等模糊表述，也不触发。只有用户明确输入 `/git-push` 时才执行。

Automate the `git add → commit → push` workflow for the current working directory.

## Core Workflow

### Phase 0: Environment Check

**Remote URL check** — determine if we'll use SSH or HTTPS:

First, check if the user provided a remote URL or if one already exists:
```bash
git remote -v 2>/dev/null
```

If no remote exists and user hasn't provided one yet, proceed to Phase 1 (will ask for URL in Phase 4).

Once a remote URL is known (user-provided or existing), check its protocol:
- **HTTPS URL** (`https://github.com/...`): skip SSH check entirely. Warn user: 「检测到 HTTPS 远程，推送时可能需要输入 GitHub 用户名和 Personal Access Token。建议改用 SSH：`git remote set-url origin git@github.com:user/repo.git`」. Proceed to Phase 1.
- **SSH URL** (`git@github.com:...`) or unknown: run SSH check:
```bash
ssh -T -o StrictHostKeyChecking=no git@github.com 2>&1
```
  - Success (exit 0 or "successfully authenticated"): proceed.
  - "Permission denied": run `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""` to generate a key, print `cat ~/.ssh/id_ed25519.pub`, and tell the user to add it at https://github.com/settings/keys. STOP — do not proceed until the user confirms.
  - If `~/.ssh/id_ed25519.pub` already exists but SSH fails, just print the key and the URL above. STOP.

### Phase 1: Status Check

```bash
git status --short
```

- **No changes**: report "没有需要提交的改动" and exit.
- **Not a git repo** (`fatal: not a git repository`): run `git init`, then `git branch -m main` to rename the default branch, then continue.
- **Rebase/merge in progress**: warn and abort.

### Phase 2: Stage Changes

Default: `git add .` (stage all changes).

If the user specifies files, stage only those:
```bash
git add <file1> <file2>
```

After staging, run `git status --short` to confirm what's staged.

### Phase 3: Commit Message

Two modes:

**Mode A — User provides message**: use it directly.
```
git commit -m "<user's message>"
```

**Mode B — AI generates message**: when the user says "自动生成"、"帮我写"、"generate" or doesn't provide a message.

1. Get the diff of staged changes:
```bash
git diff --staged --stat
git diff --staged
```

2. Analyze the changes and generate a **Conventional Commits** message in Chinese:
   - Format: `type(scope): 简要描述`
   - Types: `feat`(新功能), `fix`(修复), `refactor`(重构), `docs`(文档), `style`(样式), `chore`(杂项), `test`(测试)
   - Scope is optional, derived from the files touched
   - Keep description under 50 chars for the subject line
   - If changes are large, add a blank line then bullet-point body summarizing key changes

3. **Show the generated message to the user and ASK for confirmation** before committing.
   - **This is mandatory — do NOT commit without explicit user approval.**
   - Present the message clearly and wait for user response (y/n/edit).
   - If user says "edit" or provides alternative text, use that instead.
   - If user rejects, go back to step 1 and regenerate.
   - Only proceed to `git commit` after user explicitly approves.

### Phase 4: Remote & Push

Check remote:
```bash
git remote -v
```

- **No remote**: ask user for the GitHub repo URL (e.g., `git@github.com:user/repo.git`), then:
  1. Check URL protocol:
     - HTTPS: warn 「检测到 HTTPS 远程，推送时需要 GitHub 用户名和 Personal Access Token。建议改用 SSH 地址 `git@github.com:user/repo.git`。」, then `git remote add origin <url>`.
     - SSH: `git remote add origin <url>` directly.
  2. If the user doesn't provide a URL, print instructions and exit.
- **Remote exists but no branch name**: run `git checkout -b main` if needed (default branch should be `main`).

Push:
```bash
git push -u origin <current-branch>
```

If the push is rejected (non-fast-forward), suggest `--force-with-lease` but NEVER force-push without explicit user confirmation.

### Phase 5: Summary

Report:
- Branch pushed to
- Commit hash (short)
- Commit message
- Remote URL

## Edge Cases

- **Merge conflicts**: abort flow and tell user to resolve manually
- **Large files**: warn if any file > 50MB before staging
- **Nested git repos**: detect and handle as follows:

  1. Run `find . -name ".git" -type d | grep -v "^\./\.git$"` to find nested `.git` directories.
  2. If none found, proceed normally.
  3. If found, check `cat .gitmodules 2>/dev/null`:
     - **Has .gitmodules** (legitimate submodule): tell user:
       > 检测到子模块 `subdir` 有未提交改动。父仓库只记录子模块指针，不记录内部文件。
       > 正确流程：① 先进子模块提交推送 → ② 回到父仓库 `git add subdir` 更新指针 → ③ 父仓库提交推送。
       > 要我按这个流程处理吗？(y/n)
       If user says yes, cd into each submodule, run the full git-push flow recursively, then back to parent, `git add <submodule>`, and continue parent flow.
     - **No .gitmodules** (accidental nested repo): tell user:
       > 发现目录 `subdir` 内含 `.git`，但不是子模块。父仓库无法追踪其内部改动。
       > ① 收编为普通目录（删掉 subdir/.git，文件纳入父仓库管理）
       > ② 转为正式子模块（git submodule add）
       > ③ 跳过，我自己手动处理
       > 选哪个？(1/2/3)
       - Option 1: `rm -rf subdir/.git` then `git add subdir/`, continue parent flow.
       - Option 2: ask for remote URL, then `git submodule add <url> subdir`, continue.
       - Option 3: skip the nested repo in staging (use `git add . -- ':!subdir'` or equivalent), warn user it was excluded.
- **Detached HEAD**: warn and suggest creating a branch first
