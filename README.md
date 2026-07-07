# skills-sh-publisher

[![skills.sh](https://skills.sh/b/akirousnow/skills-sh-publisher)](https://skills.sh/akirousnow/skills-sh-publisher)

一个 Agent Skill，教你**正确地把 Skill 发布到 [skills.sh](https://skills.sh)**。

基于对 [`vercel-labs/skills`](https://github.com/vercel-labs/skills) CLI 源码的逐行实证研究，覆盖目录结构、打包机制、发布流程与常见坑——所有结论可追溯到具体源码，非主观推测。

## 为什么需要它

`npx skills add` 有一个隐蔽的坑：当 `SKILL.md` 位于仓库根目录时，CLI **只安装 SKILL.md 单个文件**，丢弃所有附加资源（指南、脚本、参考文档等）。这个 Skill 把这类坑和正确的发布姿势整理成一份可被 AI 直接调用的速查。

## 安装

```bash
npx skills add akirousnow/skills-sh-publisher
```

或克隆后手动引用：

```bash
git clone https://github.com/akirousnow/skills-sh-publisher.git
```

## 仓库结构

遵循本 Skill 自己讲解的最佳实践——`SKILL.md` 位于 `skills/<name>/` 子目录：

```
skills-sh-publisher/
├── README.md
├── .gitignore
└── skills/
    └── skills-sh-publisher/
        └── SKILL.md
```

## 致谢

- skills CLI：<https://github.com/vercel-labs/skills>
- 排行榜：<https://skills.sh>

## License

MIT
