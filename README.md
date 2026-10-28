# ginko-skill

个人 Claude Code skill 合集，用于沉淀日常工作流。

## 目录结构

```
skills/
└── <skill-name>/
    ├── SKILL.md        # 必需：frontmatter（name、description）+ 指令正文
    ├── scripts/        # 可选：可执行脚本
    ├── references/     # 可选：按需加载的参考文档
    └── assets/         # 可选：模板、图片等资源
```

## 使用方式

将需要的 skill 复制（或软链接）到 Claude Code 的 skills 目录：

```bash
# 全局可用
cp -r skills/<skill-name> ~/.claude/skills/

# 仅当前项目可用
cp -r skills/<skill-name> <project>/.claude/skills/
```

或一次性链接全部 skill：

```bash
for d in skills/*/; do ln -sfn "$PWD/$d" ~/.claude/skills/"$(basename "$d")"; done
```

## 新增 skill

1. 复制 `skills/example-skill` 为新目录，目录名与 `name` 字段保持一致（小写、连字符）。
2. 写好 `description`：说明 skill 做什么、**何时**触发——Claude 依赖它决定是否加载。
3. 正文保持精简，详细资料放到 `references/`，按需引用。

## Skill 列表

| Skill | 说明 |
| --- | --- |
| [example-skill](skills/example-skill) | 示例模板，新建 skill 时复制它 |
| [avoid-overdefensive-code](skills/avoid-overdefensive-code) | 抑制过度防御编程（多余 try/except、空值检查、静默兜底），适用于 Claude Code / Codex |
