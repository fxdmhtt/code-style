# code-style

个人代码开发规范的 skills 集合。每种语言的写作风格作为一个独立 skill，放在 `skills/` 下，按语言逐步扩展。

## 当前 skills

- [`python-development`](skills/python-development/SKILL.md) —— 偏函数式、流水线式的 Python 写法偏好（`toolz.curried`、`T.pipe`），任何涉及编写 Python 代码的任务都会触发。

## 安装

用 `gh skills` 从本仓库安装所需 skill。

## 结构

```
skills/
└── python-development/
    └── SKILL.md
```

每个 skill 一个目录，目录名即 skill 名，`SKILL.md` 内含 frontmatter（`name`、`description`）与规范正文。新增语言时，在 `skills/` 下新建对应目录即可。
