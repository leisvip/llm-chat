---
title: Git 部署工作流 — 推送效率与实战经验
description: 以 GitHub Pages 部署为例，总结 Git 推送流程中的效率技巧、常见问题与最佳实践。
tags: [Git, GitHub Pages, 部署, 工作流, 效率]
date: 2026-04-30
---

# Git 部署工作流 — 推送效率与实战经验

> 基于 LLM Chat 项目 GitHub Pages 部署的实战总结。
> 仓库：[leisvip/llm-chat](https://github.com/leisvip/llm-chat)

---

## 一、推送流程总览

### 1.1 最小推送流程（3 步）

```bash
git add -A
git commit -m "feat: 描述"
git push origin main
```

### 1.2 完整部署流程（首次）

```bash
# 1. 初始化
git init && git add -A && git commit -m "init"

# 2. 创建远程仓库并推送
gh repo create my-repo --public --source=. --remote=origin --push

# 3. 启用 GitHub Pages
gh api repos/用户/my-repo/pages -X POST \
  --input - << 'EOF'
{ "source": { "branch": "main", "path": "/" }, "build_type": "legacy" }
EOF

# 4. 等待构建
gh api repos/用户/my-repo/pages --jq '.status'
```

### 1.3 后续更新流程（日常）

```bash
git add -A && git commit -m "update: 描述" && git push origin main
# Pages 自动重建，约 30 秒 ~ 1 分钟生效
```

---

## 二、效率技巧

### 2.1 一行式提交推送

把三步合成一行，减少重复输入：

```bash
# 别名方式（推荐，在 .bashrc 或 .gitconfig 中配置）
git config --global alias.deploy '!f() { git add -A && git commit -m "$1" && git push origin main; }; f'

# 使用
git deploy "fix: 修复设定按钮"
```

```bash
# 或者直接用 && 链接
git add -A && git commit -m "fix: 修复" && git push origin main
```

### 2.2 跳过 `git add`（自动暂存）

```bash
# 配置自动暂存已跟踪文件
git config --global alias.up '!git add -u && git commit -m "$1" && git push origin main'

# 只提交已修改的文件，不包含新文件
git up "fix: 修改已有文件"
```

### 2.3 使用 `--amend` 修正最后一次提交

```bash
# 刚提交发现漏了文件或写错了描述
git add -A
git commit --amend -m "fix: 正确的描述"
git push --force origin main  # ⚠️ 仅限个人仓库
```

### 2.4 使用 `--no-verify` 跳过钩子

```bash
# 如果配置了 pre-commit 钩子但这次想跳过
git push --no-verify origin main
```

### 2.5 `gh` CLI 一键创建并推送

```bash
# 最高效的首次推送方式
gh repo create my-repo --public --source=. --remote=origin --push
# 一条命令完成：创建仓库 + 添加 remote + 推送
```

---

## 三、提交信息规范

### 3.1 格式

```
<类型>: <描述>

[可选正文]
[可选脚注]
```

### 3.2 类型速查

| 类型 | 场景 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat: 添加 CORS 代理支持` |
| `fix` | 修复 Bug | `fix: 修复引号不匹配导致 JS 解析失败` |
| `docs` | 文档 | `docs: 添加部署报告` |
| `style` | 格式调整 | `style: 统一缩进为 2 空格` |
| `refactor` | 重构 | `refactor: 提取 proxyFetch 函数` |
| `perf` | 性能优化 | `perf: 减少不必要的 DOM 操作` |
| `chore` | 杂项 | `chore: 更新 .gitignore` |

### 3.3 好的 vs 坏的提交信息

```bash
# ❌ 坏的
git commit -m "update"
git commit -m "fix bug"
git commit -m "修改"
git commit -m "asdf"

# ✅ 好的
git commit -m "fix: playNotificationSound 引号不匹配导致整个 script 块解析失败"
git commit -m "feat: 添加 DeepSeek/SiliconFlow/OpenAI 快速配置按钮"
git commit -m "fix: proxyFetch 兼容 Cloudflare Worker ?url= 参数格式"
```

### 3.4 为什么提交信息很重要

- `git log --oneline` 一目了然改了什么
- GitHub 自动生成 Release Notes
- 方便回溯和 `git bisect` 排查问题
- 团队协作时减少沟通成本

---

## 四、分支策略

### 4.1 个人项目（简单）

```bash
# 只用 main 分支，直接推送
git push origin main
```

### 4.2 需要实验时

```bash
# 创建实验分支
git checkout -b experiment/new-feature

# 实验完了合并
git checkout main
git merge experiment/new-feature
git push origin main

# 或者实验失败直接删除
git checkout main
git branch -D experiment/new-feature
```

### 4.3 多人协作（Git Flow 简化版）

```bash
main          ← 生产分支，始终保持可用
├── develop   ← 开发分支
├── feature/* ← 功能分支
└── hotfix/*  ← 紧急修复
```

---

## 五、GitHub Pages 部署专用技巧

### 5.1 `.nojekyll` 必须

```bash
# 纯静态 HTML 项目必须有这个文件，否则 GitHub 会尝试 Jekyll 构建
touch .nojekyll
```

### 5.2 Pages API 的坑

```bash
# ❌ 错误：-f 传的是字符串，不是对象
gh api repos/.../pages -X POST \
  -f source='{"branch":"main","path":"/"}'
# → 422 错误

# ✅ 正确：用 --input - 传 JSON body
gh api repos/.../pages -X POST --input - << 'EOF'
{ "source": { "branch": "main", "path": "/" } }
EOF
```

### 5.3 等待构建完成

```bash
# 方式 A：循环检查
for i in $(seq 1 12); do
  STATUS=$(gh api repos/用户/仓库/pages --jq '.status')
  [ "$STATUS" = "built" ] && break
  sleep 5
done

# 方式 B：使用 gh run watch（如果是 Actions 部署）
gh run watch
```

### 5.4 验证部署内容

```bash
# 对比本地和远程的 MD5
LOCAL=$(md5sum index.html | cut -d' ' -f1)
REMOTE=$(curl -s "https://用户.github.io/仓库/index.html" | md5sum | cut -d' ' -f1)
[ "$LOCAL" = "$REMOTE" ] && echo "✅ 一致" || echo "⚠️ 不同（CDN 缓存？）"
```

### 5.5 CDN 缓存策略

| 路径 | 缓存时间 | 建议 |
|------|----------|------|
| `/`（根路径） | 5-10 分钟 | 调试时用 `/index.html` 更快 |
| `/index.html` | 1-2 分钟 | 日常更新推荐 |
| `/assets/*` | 长时间 | 静态资源加版本号 |

---

## 六、热修复流程

当线上出 Bug 时，最高效的修复流程：

### 6.1 标准热修复流程

```bash
# 1. 拉取最新代码
git pull origin main

# 2. 定位问题（用 grep/find）
grep -n "出问题的代码" index.html

# 3. 修复（用 sed 或手动编辑）
sed -i 's/错误代码/正确代码/g' index.html

# 4. 验证修复
node -c index.html  # JavaScript 语法检查

# 5. 提交推送
git add -A && git commit -m "fix: 修复 XXX" && git push origin main

# 6. 等待构建并验证
for i in $(seq 1 12); do
  [ "$(gh api repos/.../pages --jq '.status')" = "built" ] && break
  sleep 5
done
```

### 6.2 本次实战的 3 次热修复

| 次数 | Bug | 排查方法 | 修复时间 |
|------|-----|----------|----------|
| 1 | 引号不匹配 `'llm_soundOff"` | `node -c` 语法检查 | 2 分钟 |
| 2 | HTML 模板字面量 | 目视检查 `${...}` | 5 分钟 |
| 3 | CORS 代理 URL 格式 | 阅读 `proxyFetch` 代码 | 3 分钟 |

### 6.3 排查工具箱

```bash
# JavaScript 语法检查
node -c script.js

# 查找特定代码
grep -n "关键词" index.html

# 查看文件差异
git diff

# 查看最近提交
git log --oneline -5

# 回滚到上一个版本
git revert HEAD
git push origin main

# 强制回滚（危险）
git reset --hard HEAD~1
git push --force origin main
```

---

## 七、效率对比

### 7.1 不同推送方式的效率

| 方式 | 步骤数 | 耗时 | 适合场景 |
|------|--------|------|----------|
| GUI 工具（GitHub Desktop） | 5-6 步 | 30 秒 | 不熟悉命令行 |
| 命令行三步式 | 3 步 | 15 秒 | 日常使用 |
| 一行式别名 | 1 步 | 5 秒 | 高频推送 |
| `gh repo create --push` | 1 步 | 5 秒 | 首次创建仓库 |

### 7.2 本次部署的时间分布

```
总耗时：约 1 小时 26 分钟

├─ 代码修改：30 分钟（35%）
│   ├─ 首次部署（6 处改动）：20 分钟
│   └─ 热修复（3 次）：10 分钟
│
├─ 等待构建：5 分钟（6%）
│   └─ 5 次构建，每次 30 秒 ~ 1 分钟
│
├─ 排查问题：25 分钟（29%）
│   ├─ 引号 bug 排查：15 分钟
│   └─ CORS 问题分析：10 分钟
│
├─ 文档编写：20 分钟（23%）
│   └─ 部署报告 + 本地启动脚本
│
└─ 其他（环境配置等）：6 分钟（7%）
```

---

## 八、Git 配置优化

### 8.1 推荐的全局配置

```bash
# 用户信息
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"

# 默认分支名
git config --global init.defaultBranch main

# 自动暂存已跟踪文件（git add -u 的默认行为）
git config --global push.default current

# 更友好的 diff 输出
git config --global diff.algorithm histogram

# 自动处理行尾
git config --global core.autocrlf true  # Windows
git config --global core.autocrlf input # Mac/Linux

# 别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"
git config --global alias.deploy '!f() { git add -A && git commit -m "$1" && git push origin main; }; f'
```

### 8.2 SSH vs HTTPS

| 方式 | 优点 | 缺点 |
|------|------|------|
| SSH | 免密推送，一劳永逸 | 需要配置密钥 |
| HTTPS | 简单直接 | 每次要输入密码（或用 credential helper） |

```bash
# SSH 配置（推荐）
ssh-keygen -t ed25519 -C "your@email.com"
# 把公钥添加到 GitHub → Settings → SSH Keys

# HTTPS 记住密码
git config --global credential.helper store
```

---

## 九、常见问题速查

| 问题 | 原因 | 解决 |
|------|------|------|
| `Permission denied (publickey)` | SSH 密钥未配置 | `ssh-keygen` + 添加到 GitHub |
| `Repository not found` | remote URL 错误或仓库不存在 | `gh repo create` 或 `git remote set-url` |
| `Updates were rejected` | 远程有本地没有的提交 | `git pull --rebase origin main` 再 push |
| `detached HEAD` | 在某个 commit 上游离 | `git checkout main` 回到主分支 |
| `merge conflict` | 同一行被不同提交修改 | 手动编辑冲突文件，`git add` 后 `git commit` |
| Pages 构建失败 | 文件格式问题 | 检查 `index.html` 是否在根目录 |
| Pages 404 | 未启用或分支错误 | Settings → Pages → 检查分支和目录 |
| 推送后页面没更新 | CDN 缓存 | 等 1-2 分钟，或用 `/index.html` 访问 |

---

## 十、检查清单

```
推送前
□  git status 确认变更内容        □  git diff 检查具体改动
□  JavaScript 语法检查（node -c） □  无硬编码密钥/token
□  提交信息清晰准确               □  .nojekyll 存在（Pages 项目）

推送后
□  构建状态为 built               □  页面可正常访问
□  功能验证通过                   □  无控制台报错
```
