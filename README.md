# coverage-report-skill

> **English** | [简体中文](README.zh.md)

An [opencode](https://opencode.ai) skill that adds test coverage badges to a README and deploys HTML coverage reports to GitHub Pages, using shields.io endpoint badges — no Codecov or SonarCloud required.

It is built from real experience wiring JaCoCo and Vitest coverage reports to GitHub Pages for the `oci-j/funeral` project. Every pitfall in `SKILL.md` has actually broken in production.

---

## What it does

When triggered, the skill:

1. **Inspects** the project type (Maven/Quarkus backend, Vitest/Jest frontend, etc.).
2. **Configures** the backend coverage tool (usually `jacoco-maven-plugin`).
3. **Configures** the frontend coverage tool if needed (Vitest, Jest, etc.).
4. **Updates CI** to upload raw coverage artifacts and deploy them to GitHub Pages.
5. **Merges** coverage from multiple CI jobs if necessary (e.g. unit tests + integration tests).
6. **Adds** shields.io endpoint badges to the README that link to the HTML reports.
7. **Verifies** the deployment with concrete checks (JSON endpoints, report URLs, clickability).
8. **Self-improves**: writes per-project notes and updates this skill with new pitfalls.

---

## When it triggers

The skill activates on coverage-related requests such as:

- "Add a coverage badge to README"
- "Deploy coverage reports to GitHub Pages"
- "Make the badge click through to the report"
- "Merge coverage from multiple CI jobs"
- "Don't use index.html for the coverage report"
- "quarkus-jacoco is not generating a report"
- "Coverage badge is 404"

See the full trigger list in `SKILL.md`.

---

## Quick start

Register this directory as an external skill in your `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/home/xenoamess/workspace/coverage-report-skill"]
  }
}
```

Restart opencode. Then ask, for example:

> "Add coverage badges to my project and deploy the reports to GitHub Pages."

opencode will load `SKILL.md` as system context and execute the steps inside.

---

## How it works

opencode scans each path in `skills.paths` for `**/SKILL.md` and loads the matching file as system prompt context.

- `SKILL.md` is the actual skill — strategy, pitfalls, and verification steps live there.
- `README.md` is human documentation; opencode does not read it automatically.

---

## Self-improvement loop

After a successful run the skill:

1. Writes per-project notes to `<project>/docs/coverage-report-notes.md`.
2. Updates `SKILL.md` and `README.md` with new pitfalls, trigger phrases, or tools.
3. Commits the skill changes locally.

---

## License

Apache License 2.0.
