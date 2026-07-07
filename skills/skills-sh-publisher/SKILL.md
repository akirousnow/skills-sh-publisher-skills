---
name: skills-sh-publisher
description: >-
  Step-by-step guide for correctly publishing an Agent Skill to skills.sh via
  `npx skills add`. Covers the required directory structure (SKILL.md must live
  in skills/<name>/, never at repo root), the directory-based bundling
  mechanism, two install paths (blob vs git clone), and common pitfalls like a
  root-level SKILL.md installing only a single file. Use when publishing,
  packaging, debugging, or reviewing a skill for skills.sh distribution.
  如何把 skill 正确发布到 skills.sh 的最佳实践指南，涵盖目录结构、打包机制与常见坑。
---

# 将 Skill 发布到 skills.sh 的最佳实践

> 本指南基于对 [`vercel-labs/skills`](https://github.com/vercel-labs/skills) CLI 源码的逐行实证研究，所有结论均可追溯到具体源码，非主观推测。
>
> 适用对象：想把一个 Agent Skill 通过 `npx skills add` 分发、并出现在 [skills.sh](https://skills.sh) 排行榜上的开发者。本指南**只聚焦「如何有效发布」**，不涉及 skill 内容怎么写。

---

## 0. TL;DR — 三条铁律

1. **`SKILL.md` 绝不能放在仓库根目录**——必须放在 `skills/<skill-name>/SKILL.md` 这样的子目录里，否则 `npx skills add` 只会安装 SKILL.md 单个文件，丢弃所有附加资源。
2. **不存在用于声明附加文件的 frontmatter 字段**（没有 `bundle`/`resources`/`files`/`includes`）。打包是**基于目录**的：SKILL.md 所在目录下的所有内容自动递归复制。
3. **skills.sh 没有「发布」命令**——排行榜由安装遥测驱动。你只需把仓库推到 GitHub，用户 `npx skills add owner/repo` 即可。

---

## 1. skills.sh 的运作机制

skills.sh 由两部分组成：

| 组件 | 作用 |
|------|------|
| **排行榜网站**（skills.sh） | 按匿名安装遥测数据排序，展示热门 skill |
| **CLI**（`npx skills`） | 类似 npm 的包管理器，从 GitHub / 本地路径拉取 skill |

**分发模型**：把 skill 放进一个公开 GitHub 仓库 → 用户执行 `npx skills add owner/repo` → CLI 克隆/下载并安装到 agent 的 skills 目录 → 安装事件上报遥测 → 你的 skill 出现在 skills.sh 排行榜。

> 没有 `skills publish` 这类命令。安装即上报，排行榜异步聚合。

---

## 2. 黄金法则：目录结构（最重要的坑）

### 2.1 标准结构

```
your-skill-repo/                    # GitHub 仓库根
├── README.md                       # 给人看的仓库说明（不会被装进 skill）
├── .gitignore
└── skills/                         # skill 容器目录（CLI 优先搜索）
    └── your-skill-name/            # = 一个 skill，名字须与 frontmatter name 一致
        ├── SKILL.md                # 必需，skill 入口
        ├── guide/                  # 可选，附加文档
        ├── components/             # 可选，附加资源
        ├── scripts/                # 可选，约定俗成的可执行脚本目录
        ├── references/             # 可选，约定俗成的参考文档目录
        └── assets/                 # 可选，约定俗成的静态资源目录
```

也接受单层结构 `<skill-name>/SKILL.md`（直接在仓库根下建一个以 skill 名命名的子目录），但 `skills/<name>/` 是最稳妥的，因为 `skills/` 是 CLI 的**优先搜索目录**。

### 2.2 为什么根目录放 SKILL.md 是致命错误

源码 `src/add.ts` 安装阶段有三条分支：

```js
if (blobResult && 'files' in skill) {
  // blob 路径：写入快照里的文件
} else if (tempDir && skill.path === tempDir && skill.rawContent) {
  // ⚠️ 根目录级 SKILL.md：只安装这一个文件，不打包整个仓库
  result = await installBlobSkillForAgent(
    { installName: skill.name, files: [{ path: 'SKILL.md', contents: skill.rawContent }] },
    ...
  );
} else {
  // ✅ 正常路径：copyDirectory 复制整个 skill 目录
  result = await installSkillForAgent(skill, agent, ...);
}
```

当 `skill.path === tempDir`（即 SKILL.md 在仓库根目录），CLI **故意只安装 SKILL.md 本身**，注释原话：

> `// Remote root-level SKILL.md: install the skill file, not the whole repository.`

设计意图是防止把仓库根目录里的无关文件（README、CI 配置等）打包进 skill。代价是：**根目录 SKILL.md 的所有同级资源（guide/、components/ 等）全部丢失**。

**实测影响**：一个含 SKILL.md + 7 篇 guide + 57 个组件的 skill，根目录布局下安装只得到 1 个文件；移入 `skills/<name>/` 后安装得到 65 个文件。

---

## 3. 打包机制详解

### 3.1 复制规则（`copyDirectory`）

源码 `src/installer.ts`：

```js
const EXCLUDE_FILES = new Set(['metadata.json']);
const EXCLUDE_DIRS = new Set(['.git', '__pycache__', '__pypackages__']);
```

`copyDirectory(skill.path, dest)` **递归复制 skill 目录下的全部内容**，仅排除上述四项。所以：

- 任意子目录、任意扩展名的文件都会被复制
- 无需在任何地方声明它们
- `.md`、`.sh`、`.py`、图片、JSON 等一律自动带上

### 3.2 两种安装路径

| 路径 | 触发条件 | 数据来源 |
|------|----------|----------|
| **blob 快速路径** | repo owner ∈ `['vercel','vercel-labs','heygen-com']` 或在 `BLOB_ALLOWED_REPOS` 名单 | `skills.sh/api/download/{owner}/{repo}/{slug}` 预构建快照 |
| **git clone 路径** | 其他所有 repo（绝大多数个人/第三方 skill） | `git clone --depth=1` → `discoverSkills` → `copyDirectory` |

> 普通开发者的 skill 走的是 **git clone 路径**。blob 路径是 Vercel 系仓库的专属加速通道。这意味着 skills.sh 网站上的快照显示（异步缓存）与实际安装结果可能短暂不一致，以**实际安装结果为准**。

---

## 4. 发布流程

### 4.1 准备

```bash
# 1. 确认目录结构正确（SKILL.md 不在根目录）
ls skills/your-skill-name/SKILL.md   # 必须存在

# 2. 确认 frontmatter 合法（至少有 name 和 description）
head -5 skills/your-skill-name/SKILL.md
```

### 4.2 推送到 GitHub

```bash
git add -A
git commit -m "feat: initial skill"
git push origin main
```

仓库须为**公开**（private repo 需要凭证，且 blob 路径无法服务）。

### 4.3 验证可发现性（不安装）

```bash
npx skills add owner/repo --list
```

应列出你的 skill 及其 description。这一步只克隆+解析，不写入 agent 目录。

### 4.4 真实安装测试

```bash
# 全局安装到 Cursor（-g），避免污染当前项目
npx skills add owner/repo -g -a cursor -y
```

### 4.5 检查安装结果（关键！）

```bash
# 列出实际安装的文件，确认附加资源都在
ls -R ~/.agents/skills/your-skill-name/
```

> 如果只看到 SKILL.md、没有子目录 → 回到第 2 节，检查 SKILL.md 是否误放在仓库根目录。

### 4.6 加徽章到 README

```markdown
[![skills.sh](https://skills.sh/b/owner/repo)](https://skills.sh/owner/repo)
```

把 `owner/repo` 换成你的仓库。

---

## 5. 验证清单

发布前逐项确认：

- [ ] `SKILL.md` 位于 `skills/<name>/` 子目录，**不在仓库根**
- [ ] frontmatter 有合法的 `name`（kebab-case）和 `description`
- [ ] `name` 与目录名 `<name>` 一致
- [ ] 附加资源（文档/脚本）与 SKILL.md 同级或在其子目录
- [ ] 仓库为公开
- [ ] `npx skills add owner/repo --list` 能发现 skill
- [ ] 真实安装后 `ls -R` 确认所有文件到位
- [ ] README 已加 skills.sh 徽章
- [ ] `skills-lock.json` 已加入 `.gitignore`，未提交

---

## 6. 常见坑与排错

### 坑 1：安装后只有 SKILL.md，附加文件全丢

**原因**：SKILL.md 放在了仓库根目录，触发 `add.ts` 的单文件安装分支。
**修复**：把 SKILL.md 连同所有资源移入 `skills/<skill-name>/` 子目录。

### 坑 2：`skills-lock.json` 被误提交

**原因**：在项目目录内运行 `npx skills add`（非 `-g`）会生成项目级 lock 文件。
**修复**：`git rm --cached skills-lock.json`，加入 `.gitignore`。

### 坑 3：skills.sh 网站快照与实际安装不一致

**原因**：网站快照是异步缓存的；且 blob 快照路径仅对特定 owner 生效。
**说明**：以**实际 `npx skills add` 安装结果**为准，网站显示会随后刷新。

### 坑 4：skill 名含空格或大写

**原因**：`name` 须 kebab-case。slug 计算（`toSkillSlug`）会强制小写、连字符化，与目录名不一致可能导致发现/安装错位。
**修复**：name、目录名统一用 kebab-case（如 `my-skill`）。

### 坑 5：仓库名拼写错误

**原因**：remote URL 是安装入口，拼写错误（如 `componenets`）会固化到所有文档和徽章里。
**修复**：尽早改名，GitHub 仓库改名会自动重定向旧 URL，但 README/徽章里的引用要同步更新。

### 坑 6：private repo 安装失败

**原因**：git clone 需要 HTTPS 凭证或 SSH key。
**修复**：`gh auth login`，或用 SSH URL `npx skills add git@github.com:owner/repo.git`。

---

## 7. 进阶：本地开发与多 skill 仓库

### 本地路径安装（开发期快速迭代）

```bash
npx skills add ./path/to/skill-parent-dir
```

指向包含 skill 目录的父目录即可，CLI 会用 `discoverSkills` 扫描。

### 一个仓库放多个 skill

```
multi-skill-repo/
└── skills/
    ├── skill-a/
    │   └── SKILL.md
    └── skill-b/
        └── SKILL.md
```

`npx skills add owner/repo --list` 会列出全部；`--skill skill-a` 可指定单个。

### 全局 vs 项目级安装

| 范围 | 命令 | 位置 | 场景 |
|------|------|------|------|
| 项目 | `npx skills add owner/repo` | `./.agents/skills/` | 团队共享、随项目提交 |
| 全局 | `npx skills add owner/repo -g` | `~/.agents/skills/` | 跨项目复用 |

---

## 8. CLI 常用命令速查

```bash
npx skills add owner/repo              # 安装
npx skills add owner/repo --list       # 仅列出，不安装（验证发现）
npx skills add owner/repo -g -a cursor -y   # 全局装到 Cursor，跳过确认
npx skills list                        # 列出已装的 skill
npx skills list -g                     # 列出全局 skill
npx skills find <query>                # 搜索 skill
npx skills update                      # 更新已装 skill
npx skills remove <name>               # 卸载
npx skills init [name]                 # 生成 SKILL.md 模板
npx skills use owner/repo              # 不安装，临时生成 prompt 使用
```

---

## 9. 参考资源

- CLI 源码：<https://github.com/vercel-labs/skills>
- 排行榜：<https://skills.sh>
- Skill 规范：<https://agentskills.io/specification>
- 官方示例仓库：<https://github.com/vercel-labs/agent-skills>

---

*本指南所有技术结论均来自 `vercel-labs/skills` 源码实证（`add.ts` / `installer.ts` / `skills.ts` / `blob.ts` / `git.ts`）。若 CLI 升级导致行为变化，以最新源码为准。*
