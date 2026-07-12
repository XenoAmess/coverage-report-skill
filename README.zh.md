# coverage-report-skill

> [English](README.md) | **简体中文**

一个 [opencode](https://opencode.ai) 技能（skill），用于为项目 README 添加测试覆盖率徽章，并将 HTML 覆盖率报告部署到 GitHub Pages。使用 shields.io endpoint badge，无需 Codecov / SonarCloud。

它源自为 `oci-j/funeral` 项目把 JaCoCo 和 Vitest 覆盖率报告接到 GitHub Pages 的真实经验。`SKILL.md` 里列出的每一个坑都是生产环境中真实踩过的。

---

## 它做什么

被触发时，该技能会：

1. **检查**项目类型（Maven/Quarkus 后端、Vitest/Jest 前端等）。
2. **配置**后端覆盖率工具（通常是 `jacoco-maven-plugin`）。
3. 按需**配置**前端覆盖率工具（Vitest、Jest 等）。
4. **更新 CI**：上传原始覆盖率 artifact 并部署到 GitHub Pages。
5. 如有需要，**合并**多个 CI job 的覆盖率（例如单元测试 + 集成测试）。
6. 在 README 里**添加** shields.io endpoint 徽章，并链接到 HTML 报告。
7. 用具体检查项**验证**部署是否成功（JSON 端点、报告 URL、可点击性）。
8. **自我进化**：写入项目级笔记，并把新踩的坑回写到技能本身。

---

## 触发场景

该技能在遇到以下覆盖率相关请求时激活：

- “给 README 加覆盖率徽章”
- “把覆盖率报告部署到 GitHub Pages”
- “badge 点击后跳转到报告”
- “合并多个 CI job 的覆盖率”
- “覆盖率报告不要用 index.html”
- “quarkus-jacoco 不生成报告”
- “覆盖率 badge 404”

完整触发列表见 `SKILL.md`。

---

## 快速开始

把本目录作为外部技能注册到 `opencode.json`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/home/xenoamess/workspace/coverage-report-skill"]
  }
}
```

重启 opencode，然后说例如：

> “给我的项目加覆盖率徽章，并把报告部署到 GitHub Pages。”

opencode 会把 `SKILL.md` 作为系统上下文加载，并按其中的步骤执行。

---

## 底层原理

opencode 会扫描 `skills.paths` 中每个路径下的 `**/SKILL.md`，并把匹配到的文件作为系统提示上下文加载。

- `SKILL.md` 才是真正的技能——策略、坑点和校验步骤都在那里。
- `README.md` 只是给人类看的文档，opencode 不会自动读取。

---

## 自我进化循环

每次成功执行后，本技能会：

1. 把项目笔记写到 `<project>/docs/coverage-report-notes.md`。
2. 用新的坑点、触发词或工具更新 `SKILL.md` 和 `README.md`。
3. 在本地提交技能修改。

---

## 许可证

Apache License 2.0。
