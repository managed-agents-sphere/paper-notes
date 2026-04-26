# CLAUDE.md

`managed-agents-sphere/paper-notes` 的 Claude 操作手册。**只记从 README 和代码里看不出来的东西**：路径、网络坑、过去的失误。规范、tag 表、段落结构请读 [README.md](./README.md) 和 [templates/paper.md](./templates/paper.md)。

---

## 关键事实（已踩过的坑，不要再犯）

- **org 名是 `managed-agents-sphere`**（agent 后面有 `s`，复数）。曾误写为单数 `managed-agent-sphere`，导致本地目录名与 GitHub 不一致
- **必须用 SSH remote**：`git@github.com:managed-agents-sphere/paper-notes.git`。HTTPS push 会超时（用户网络 → github.com:443 不稳），不要切回去
- **arxiv.org 在 WebFetch 里被拦**，抓论文用 `curl -sL -A "Mozilla/5.0" https://arxiv.org/pdf/<id> -o /tmp/<slug>.pdf`，然后 `Read` PDF
- **本地路径**：`/Users/ax/go/src/github.com/managed-agents-sphere/paper-notes/`
- **默认分支**：`main`；可见性：public

---

## 加一篇新论文（标准动作）

```bash
cd /Users/ax/go/src/github.com/managed-agents-sphere/paper-notes
cp templates/paper.md papers/<arxiv-id>-<slug>.md
# 写完笔记后：
# 1. 回 README.md 的"论文索引"表加一行
# 2. 回 README.md 的"最近添加"加一条
git add . && git commit -m "add: <slug> (arXiv <id>)" && git push
```

**最容易遗漏第 1、2 步**——索引不更新等于这篇论文不存在。push 前自检：

- [ ] README 索引表加了一行
- [ ] README "最近添加"加了一条
- [ ] 笔记里所有 tag 都在 README 受控词表内
- [ ] frontmatter 的 `verdict` 是**判断句**不是描述句

---

## commit message 约定

- `add: <slug> (arXiv <id>)` — 新论文
- `update: <slug> — <改动>` — 修订已有
- `chore: <什么>` — 模板/规范/CLAUDE.md 改动
- `docs: <什么>` — README/索引/聚合页

带 Claude 协作时附：
```
Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

---

## 不要做

- ❌ 不要切回 HTTPS remote
- ❌ 不要把 PDF 提交进 repo（用 arxiv 链接归档；要离线归档再加 git-lfs）
- ❌ 不要为单篇论文建子文件夹（保持扁平 `papers/<slug>.md`）
- ❌ 不要用 `git add -A`（避免误带入 `/tmp/` 等路径）
- ❌ 不要 `git push --force`
- ❌ 不要新建 `docs/` `mkdocs.yml` `quartz.config.ts` 等渲染层（阶段一只用纯 markdown）
- ❌ 不要在 CLAUDE.md 里重复 README 已有内容（受众不同，但 Claude 同一会话两个都读，重复浪费 context）

---

## 未来再做（≥5 篇论文后）

- Quartz 4 渲染静态站，激活 `[[wiki-link]]` 双链
- 脚本从 frontmatter 自动生成 README 索引表
- GitHub Actions 检查 frontmatter + tag 合法性
- git-lfs 归档原文 PDF
