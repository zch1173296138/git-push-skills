# Git Push Skill

Open Clacky 平台的 Git 推送技能 —— 一句话完成 `add → commit → push` 全流程。

## 功能

- **一键推送**：`/git-push` 自动完成暂存、提交、推送
- **AI 生成 Commit Message**：分析 diff 自动生成符合 [Conventional Commits](https://www.conventionalcommits.org/) 规范的中文提交信息
- **SSH 自动检测**：检测 SSH 认证状态，未配置时生成密钥并引导添加
- **HTTPS 兼容**：支持 Personal Access Token 推送，并给出安全提示
- **嵌套仓库处理**：自动检测子目录中的 `.git`，区分子模块和误嵌套，分别提供处理方案
- **安全检查**：冲突检测、大文件警告、禁止无确认推送

## 安装

### 方式一：从 GitHub 安装

将仓库克隆到 Clacky 的 skills 目录：

```bash
git clone https://github.com/zch1173296138/git-push-skills.git ~/.clacky/skills/git-push/
```

重启 Clacky 即可。

### 方式二：从 ZIP 包安装

1. 下载 [git-push-skill.zip](https://github.com/zch1173296138/git-push-skills/releases/latest)（或从 Releases 获取）
2. 在 Clacky 中执行 `/skill-add <zip 路径>`

### 方式三：直接下载文件

将本仓库中的 `SKILL.md` 和 `evals/` 目录放入 `~/.clacky/skills/git-push/` 即可。

## 使用

在 Clacky 对话中输入：

```
/git-push
```

或带自定义 commit message：

```
/git-push fix: 修复登录页面样式错乱
```

## 触发规则

**此 skill 仅响应用户显式的 `/git-push` 斜杠命令**，不会自动触发。即使用户说「代码写完了」「帮我保存到 git」等模糊表述，也不会激活此 skill。这确保你不会意外推送未准备好的代码。

## 工作流程

```
Phase 0  环境检查 → SSH/HTTPS 认证检测
Phase 1  状态检查 → git status，空改动则退出
Phase 2  暂存改动 → git add .
Phase 3  提交信息 → 用户指定 or AI 分析 diff 生成（需用户确认）
Phase 4  推送到远程 → git push -u origin <branch>
Phase 5  摘要报告 → 分支、commit hash、commit message
```

### Commit Message 生成

AI 模式会读取 `git diff --staged` 并生成符合 Conventional Commits 规范的中文提交信息：

```
feat(git-push): 新增嵌套仓库自动检测功能

- 自动扫描子目录中的 .git
- 区分子模块与误嵌套场景
- 分别提供处理选项
```

生成后会展示给你确认，**未确认不会提交**。

## 边界情况处理

| 场景 | 处理 |
|------|------|
| 无改动 | 报告并退出，不执行任何操作 |
| 非 Git 仓库 | 自动 `git init`，分支改名 `main` |
| SSH 未配置 | 生成 ed25519 密钥对，引导添加公钥 |
| Merge 冲突中 | 中止流程，提示手动解决 |
| 大文件（>50MB） | 暂存前警告 |
| 嵌套 `.git`（子模块） | 引导先提交子模块，再更新父仓库指针 |
| 嵌套 `.git`（误嵌套） | 提供收编/转子模块/跳过 三种选项 |
| Detached HEAD | 警告并建议创建分支 |
| Push 被拒 | 建议 `--force-with-lease`，绝不自动强推 |

## 评测

`evals/evals.json` 包含 3 个评测用例，覆盖：

1. 用户指定 commit message 的完整流程
2. AI 自动生成 commit message 的流程
3. 无改动时的空操作场景

## 文件结构

```
git-push-skill/
├── SKILL.md          # Skill 定义（核心逻辑）
├── evals/
│   └── evals.json    # 评测用例
└── README.md         # 本文件
```

## 许可证

MIT
