# code-style

个人开发规范的 skills 集合。不同语言与开发领域的规范作为独立 skill，统一放在 `skills/` 下。

## 当前 skills

- [`python-development`](skills/python-development/SKILL.md) —— 偏函数式、流水线式的 Python 写法偏好（`toolz.curried`、`T.pipe`），任何涉及 Python 代码的任务都会触发。
- [`rust-development`](skills/rust-development/SKILL.md) —— 表达式优先、迭代器链驱动的 Rust 写法（nightly-first），任何涉及 Rust 代码的任务都会触发。
- [`3d-modeling-development`](skills/3d-modeling-development/SKILL.md) —— 约束驱动、精确STEP交付与回读验证的工程3D建模规范，适用于工程功能模型。

## 安装

用 `gh skills` 从本仓库安装所需 skill。

## 结构

```
skills/
├── 3d-modeling-development/
│   └── SKILL.md
├── python-development/
│   └── SKILL.md
└── rust-development/
    └── SKILL.md
```

每个 skill 一个目录，目录名即 skill 名，`SKILL.md` 内含 frontmatter（`name`、`description`）与规范正文。新增规范时，在 `skills/` 下新建对应目录即可。
