---
name: coverage-report-skill
description: 为仓库生成测试覆盖率徽章（shields.io endpoint badge）并部署 HTML 报告到 GitHub Pages。触发场景：用户说“添加覆盖率 badge / 徽章”、“把覆盖率报告挂到 GitHub Pages”、“badge 点击后跳转报告”、“合并多个 CI job 的覆盖率”、“覆盖率报告不要 index.html”、“不要占用 index.html”、“quarkus-jacoco 不生成报告”、“JaCoCo 报告为空”、“前端覆盖率没上传”、“覆盖率 badge 404 / 不更新”。Built from wiring JaCoCo + Vitest coverage reports to GitHub Pages for oci-j/funeral.
---

# Coverage Report Skill

This skill adds a test-coverage badge to a project’s README and deploys the full HTML coverage reports to GitHub Pages, using shields.io endpoint badges so no external SaaS (Codecov/SonarCloud) is required.

It is built from real experience. The pitfalls and verification steps below are things that have actually failed.

---

## When to use

Use this skill when the user says any of:

- "添加覆盖率 badge / 徽章"
- "把覆盖率报告挂到 GitHub Pages"
- "badge 点击后跳转报告"
- "合并多个 CI job 的覆盖率"
- "覆盖率报告不要 index.html"
- "不要占用 index.html"
- "coverage.html"
- "quarkus-jacoco 不生成报告"
- "JaCoCo 报告为空"
- "前端覆盖率没上传"
- "覆盖率 badge 404 / 不更新"
- "coverage report to Pages"
- "coverage badge in README"

---

## Pre-flight checks

Before touching anything, gather:

1. **What language / build tool is used?**
   - Java backend? Check if `pom.xml` exists.
   - Quarkus project? Check if `quarkus-jacoco` is already present or if `io.quarkus` dependencies exist.
   - JavaScript/TypeScript frontend? Check `package.json` and test runner (`vitest`, `jest`, etc.).
2. **Which CI workflow runs tests and uploads artifacts?**
   - Usually `.github/workflows/build.yml`.
3. **Is GitHub Pages enabled?**
   - Use `gh api repos/<owner>/<repo>/pages` to check.
   - If 404, Pages is not enabled yet. The skill will enable it.
4. **What is the README filename?**
   - GitHub displays `README.md`. If the repo has `readme.md`, rename it to `README.md` first.

---

## Strategy

1. Generate HTML coverage reports in CI.
2. Upload the reports as GitHub Actions artifacts.
3. (Optional) If multiple jobs produce coverage, merge raw `jacoco.exec` files before generating the final report.
4. Deploy the reports to GitHub Pages via a dedicated job.
5. Add shields.io endpoint badges to README that point to the Pages URLs.

---

## Implementation

### Step 1 — Backend coverage (Maven / Java)

For plain JUnit or Quarkus projects, prefer `jacoco-maven-plugin` over `quarkus-jacoco`.

**Why**: `quarkus-jacoco` only instruments `@QuarkusTest` tests. If the CI runs plain JUnit tests, the report will be empty.

Add to `pom.xml`:

```xml
<properties>
    <jacoco.version>0.8.15</jacoco.version>
</properties>
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>${jacoco.version}</version>
            <executions>
                <execution>
                    <id>prepare-agent</id>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/jacoco-report</outputDirectory>
                    </configuration>
                </execution>
                <execution>
                    <id>merge-coverage</id>
                    <goals>
                        <goal>merge</goal>
                    </goals>
                    <configuration>
                        <destFile>${project.build.directory}/merged.exec</destFile>
                        <fileSets>
                            <fileSet>
                                <directory>${project.build.directory}</directory>
                                <includes>
                                    <include>jacoco*.exec</include>
                                </includes>
                            </fileSet>
                        </fileSets>
                    </configuration>
                </execution>
                <execution>
                    <id>report-merged</id>
                    <goals>
                        <goal>report</goal>
                    </goals>
                    <configuration>
                        <dataFile>${project.build.directory}/merged.exec</dataFile>
                        <outputDirectory>${project.build.directory}/jacoco-merged-report</outputDirectory>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Run `mvn -B verify` to generate the report.

---

### Step 2 — Frontend coverage (Vitest)

If the project uses `vitest`, coverage is already configured if `@vitest/coverage-v8` is installed.

Ensure `vite.config.js` / `vitest.config.js` has:

```js
export default {
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'json-summary'],
      reportsDirectory: './coverage',
    },
  },
};
```

Run `pnpm run test:coverage` or `npm run test:coverage`.

The HTML report is in `coverage/index.html`, and the summary is in `coverage/coverage-summary.json`.

---

### Step 3 — Upload coverage artifacts

In the job that runs tests, upload the raw reports:

```yaml
- name: Upload backend jacoco exec
  if: always()
  uses: actions/upload-artifact@v7
  with:
    name: backend-jacoco-exec-${{ github.sha }}
    path: funeral-backend/target/jacoco.exec
    retention-days: 1

- name: Upload frontend coverage
  if: always()
  uses: actions/upload-artifact@v7
  with:
    name: frontend-coverage-${{ github.sha }}
    path: funeral-frontend/coverage/
    retention-days: 7
```

For the backend, upload the raw `jacoco.exec`, not the HTML report, so another job can merge it.

---

### Step 4 — Merge coverage from multiple jobs (optional)

If one job runs unit tests and another runs integration tests, make the integration job depend on the unit-test job, download its `jacoco.exec`, and merge:

```yaml
containerd-image-store-integration:
  name: Containerd Image Store Integration Test
  runs-on: ubuntu-latest
  needs: build-jvm

  steps:
    # ... setup, run integration tests ...

    - name: Download backend jacoco exec
      uses: actions/download-artifact@v8
      with:
        name: backend-jacoco-exec-${{ github.sha }}
        path: /tmp/backend-jacoco-exec

    - name: Merge coverage and generate report
      working-directory: ./funeral-backend
      run: |
        mv /tmp/backend-jacoco-exec/jacoco.exec target/jacoco-backend.exec
        mvn -B jacoco:merge@merge-coverage jacoco:report@report-merged

    - name: Upload merged backend coverage report
      if: always()
      uses: actions/upload-artifact@v7
      with:
        name: backend-coverage-${{ github.sha }}
        path: funeral-backend/target/jacoco-merged-report/
        retention-days: 7
```

---

### Step 5 — Deploy to GitHub Pages

Add a job that:

1. Downloads backend and frontend coverage artifacts.
2. Generates two JSON files for shields.io.
3. Copies the reports into `pages/backend` and `pages/frontend`.
4. Renames `index.html` to `coverage.html` so the root `index.html` slot is not occupied.
5. Deploys `pages/` to GitHub Pages.

```yaml
coverage-pages:
  name: Deploy Coverage Badges to GitHub Pages
  runs-on: ubuntu-latest
  needs: [build-jvm, containerd-image-store-integration]
  if: github.ref == 'refs/heads/main'
  permissions:
    contents: read
    pages: write
    id-token: write
  environment:
    name: github-pages
  steps:
    - uses: actions/checkout@v7

    - name: Download backend coverage
      uses: actions/download-artifact@v8
      with:
        name: backend-coverage-${{ github.sha }}
        path: backend-coverage

    - name: Download frontend coverage
      uses: actions/download-artifact@v8
      with:
        name: frontend-coverage-${{ github.sha }}
        path: frontend-coverage

    - name: Generate coverage badge JSON
      run: |
        mkdir -p pages
        python - <<'PY'
        import json
        import xml.etree.ElementTree as ET

        # backend
        root = ET.parse('backend-coverage/jacoco.xml').getroot()
        for c in root.findall('counter'):
            if c.get('type') == 'INSTRUCTION':
                covered = int(c.get('covered'))
                missed = int(c.get('missed'))
                pct = round(covered / (covered + missed) * 100)
                break
        else:
            pct = 0
        color = 'green' if pct >= 80 else 'yellow' if pct >= 60 else 'red'
        with open('pages/coverage-backend.json', 'w') as f:
            json.dump({
                "schemaVersion": 1,
                "label": "backend coverage",
                "message": f"{pct}%",
                "color": color
            }, f)

        # frontend
        with open('frontend-coverage/coverage-summary.json') as f:
            summary = json.load(f)
        total = summary['total']
        pct = round(sum(total[k]['pct'] for k in ['lines', 'statements', 'functions', 'branches']) / 4)
        color = 'green' if pct >= 80 else 'yellow' if pct >= 60 else 'red'
        with open('pages/coverage-frontend.json', 'w') as f:
            json.dump({
                "schemaVersion": 1,
                "label": "frontend coverage",
                "message": f"{pct}%",
                "color": color
            }, f)
        PY
        mkdir -p pages/backend
        cp -R backend-coverage/* pages/backend/
        mv pages/backend/index.html pages/backend/coverage.html
        mkdir -p pages/frontend
        cp -R frontend-coverage/* pages/frontend/
        mv pages/frontend/index.html pages/frontend/coverage.html

    - name: Upload Pages artifact
      uses: actions/upload-pages-artifact@v3
      with:
        path: pages/

    - name: Deploy to GitHub Pages
      uses: actions/deploy-pages@v4
```

Enable Pages via GitHub API if needed:

```bash
gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow
```

---

### Step 6 — README badge with link

```markdown
[![Backend Coverage](https://img.shields.io/endpoint?url=https://<owner>.github.io/<repo>/coverage-backend.json)](https://<owner>.github.io/<repo>/backend/coverage.html)
[![Frontend Coverage](https://img.shields.io/endpoint?url=https://<owner>.github.io/<repo>/coverage-frontend.json)](https://<owner>.github.io/<repo>/frontend/coverage.html)
```

---

## Pitfalls

### Pitfall 1 — `quarkus-jacoco` does not cover plain JUnit tests

**Symptom**: `target/jacoco-report/` is empty after `mvn test`.

**Cause**: The Quarkus JaCoCo extension only instruments `@QuarkusTest` classes. If the CI runs plain JUnit tests, the exec file is empty.

**Fix**: Use `jacoco-maven-plugin` directly.

### Pitfall 2 — `mvn test` does not generate the Quarkus JaCoCo report

**Symptom**: Report directory missing after `mvn test -Dtest=...`.

**Cause**: The Quarkus extension writes the report during the Quarkus build phase, which is part of `package`/`verify`, not `test`.

**Fix**: Run `mvn verify` instead of `mvn test` when you need the report.

### Pitfall 3 — Artifact name mismatch

**Symptom**: `coverage-pages` job fails with "Artifact not found".

**Cause**: `upload-artifact` and `download-artifact` names do not match, or the artifact was uploaded by a job not in `needs`.

**Fix**: Use the same `${{ github.sha }}` suffix and ensure `needs:` includes the producing job.

### Pitfall 4 — GitHub Pages not enabled

**Symptom**: Badge image is broken / JSON URL 404.

**Cause**: Pages is not enabled or Source is not set to "GitHub Actions".

**Fix**: Enable Pages via API:

```bash
gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow
```

### Pitfall 5 — Shields.io caches the badge

**Symptom**: Badge does not update after coverage changes.

**Cause**: shields.io caches endpoint badges for a few minutes.

**Fix**: Wait, or add a cache-buster query parameter (e.g. `?t=<timestamp>`). Usually not necessary.

### Pitfall 6 — Badge is not a link

**Symptom**: Clicking the badge does nothing.

**Cause**: The Markdown image is not wrapped in a link.

**Fix**: Use `[![alt](image-url)](link-url)`.

### Pitfall 7 — Coverage report occupies `index.html`

**Symptom**: User wants `index.html` reserved for something else.

**Fix**: Rename the report entry to `coverage.html` and do not keep `index.html`.

### Pitfall 8 — `download-artifact` path is treated as a directory

**Symptom**: Downloaded file lands at an unexpected path.

**Cause**: The `path` input of `actions/download-artifact` is a directory, even for single-file artifacts.

**Fix**: Download to a temp directory, then `mv` the file to the desired location.

### Pitfall 9 — Coverage percentage is not the same as the report shows

**Symptom**: Badge shows 26% but the HTML report shows 25%.

**Cause**: The badge parses the raw `INSTRUCTION` counter, while the HTML report may round differently.

**Fix**: This is expected. Pick one counter for the badge and document it.

### Pitfall 10 — Coverage workflow fails on main after a dependency merge

**Symptom**: All CI checks passed on the PR branch, but after merging to `main` the `coverage-pages` job fails.

**Cause**: The merged state on `main` is different from the PR head (e.g., a frontend build-tool major-version bump changes the coverage report layout, or a CI workflow change on `main` is not reflected in the PR branch).

**Fix**: After merging, watch the first `main` run and verify `coverage-pages` succeeds. For PRs that touch frontend tooling (Vite, Vitest, etc.), update the branch to include latest `main` before merging so the CI run is representative.


## Verification

After the first CI run:

1. `https://<owner>.github.io/<repo>/coverage-backend.json` returns JSON.
2. `https://<owner>.github.io/<repo>/backend/coverage.html` shows the JaCoCo report.
3. `https://<owner>.github.io/<repo>/frontend/coverage.html` shows the Vitest report.
4. `https://<owner>.github.io/<repo>/backend/index.html` returns 404 if the user requested the slot freed.
5. README badges are clickable and lead to the reports.
6. After merging a PR that touches frontend tooling or CI, verify the next `main` run’s `coverage-pages` job succeeds.

---

## Self-improvement

After each successful run:

1. Write per-project notes to `<project>/docs/coverage-report-notes.md` summarizing:
   - backend tool (JaCoCo / Quarkus / etc.)
   - frontend tool (Vitest / Jest / etc.)
   - whether multiple coverage jobs were merged
   - any pitfalls hit
2. Update this `SKILL.md` with new trigger phrases, pitfalls, or fixes.
3. Commit the skill changes locally.

---

## Worked example: oci-j/funeral

- Backend: Quarkus + Maven; used `jacoco-maven-plugin` after discovering `quarkus-jacoco` could not cover plain JUnit tests.
- Frontend: Vitest + `@vitest/coverage-v8`.
- Coverage merged from `build-jvm` (unit tests) and `containerd-image-store-integration` (integration tests).
- Deployed to `https://oci-j.github.io/funeral/`.
- Badges wrapped in Markdown links.
- Report entry renamed to `coverage.html` to free the `index.html` slot.
- After merging PR #28 (Vite 5.4.21 → 8.1.4), the next `main` run’s `coverage-pages` job succeeded, confirming the frontend coverage pipeline still worked.
