# Spectra Report

**Playwright test reports that tell you *why* tests failed, not just *which* ones.**

Spectra groups failures by root cause, adds an AI hint with the likely cause and a suggested fix for each group, and publishes a fast static report to GitHub Pages. On pull requests it posts one sticky comment that separates new failures from known ones.

- 🧩 **Failure groups.** 40 red tests with one broken locator show up as one group with 40 tests, so you fix the top group first.
- ✦ **AI root-cause hints (optional).** One model call per *new* group, cached by signature, so recurring failures cost nothing. Works with Anthropic, OpenAI or any OpenAI-compatible endpoint (Azure OpenAI, Ollama). Secrets are redacted before anything is sent.
- 🆕 **New vs known failures.** History is stored on `gh-pages`, so every test shows its last 20 results and PRs are compared with the base branch.
- 🔁 **Flaky detection.** Flake rate per test, plus a copyable quarantine list.
- 💬 **Meets developers on the PR.** A sticky PR comment and a summary on the Actions run page, both linking to the full report.
- 📦 **Zero infrastructure.** One static HTML page, with no server, database or CDN. It opens from GitHub Pages, a CI artifact or your disk.
- 🌗 **Dark mode, keyboard navigation, deep links** for every filter and test.

![Failure groups with AI hints](docs/failure-groups.png)

> Status: **v0.1 MVP**. Playwright is fully supported; any other framework works at a basic level through JUnit XML.

---

## Quick start

### 1. Add the reporter

```bash
npm i -D github:surkuar/spectra-report
```

> Spectra is not on npm yet, so install it from GitHub as shown. An npm release (`npm i -D spectra-report`) is planned.

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  retries: process.env.CI ? 1 : 0,           // retries let Spectra detect flaky tests
  use: { screenshot: 'only-on-failure', trace: 'retain-on-failure' },
  reporter: [
    ['list'],
    ['spectra-report/playwright', { outputDir: 'spectra-results' }],
  ],
});
```

### 2. Add the Action

```yaml
# .github/workflows/e2e.yml
name: e2e
on: [push, pull_request]

permissions:
  contents: write        # publish to gh-pages
  pull-requests: write   # PR comment

concurrency:
  group: spectra-${{ github.ref }}
  cancel-in-progress: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test

      - name: Spectra Report
        if: always()                      # report on red runs too
        uses: surkuar/spectra-report@v0
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}   # optional: AI hints
```

### 3. Turn on GitHub Pages (once)

In **Settings → Pages**, set **Source: Deploy from a branch** and choose **`gh-pages` / root**. The branch is created on the first run.

Reports are published at:

| What | URL |
| --- | --- |
| Latest report for a branch | `https://<owner>.github.io/<repo>/<branch>/latest/` |
| A specific run | `https://<owner>.github.io/<repo>/<branch>/<run-number>/` |
| All branches | `https://<owner>.github.io/<repo>/` |

> ⚠️ On a **public** repo the Pages site is public, and that includes screenshots and traces. If your tests touch real customer data, set `publish: false` and upload `spectra-report/` as a workflow artifact instead.

---

## Local use

```bash
npx playwright test                 # writes spectra-results/
npx spectra generate                # builds spectra-report/
open spectra-report/index.html      # or: npx serve spectra-report
```

Keep history between local runs:

```bash
npx spectra generate --history spectra-report/history.json --history-out .spectra-history.json
```

### CLI

```
spectra generate [options]
  -r, --results <dir>      Folder written by the Playwright reporter (repeatable for shards)
  -j, --junit <file>       JUnit XML to import: TestNG, JUnit, pytest, Cucumber-JVM, Jest... (repeatable)
  -o, --out <dir>          Output folder (default: spectra-report)
      --history <file>     history.json from the previous run
      --history-out <file> Where to write the updated history
      --report-url <url>   Public URL, used for links in the summary
      --name <text>        Run name shown in the header
      --no-ai              Disable AI even if a key is set

spectra summary <reportDir> Print the markdown summary
```

### Sharded runs

Give each shard its own results folder, then pass all of them. Spectra merges them into one report:

```bash
SPECTRA_RESULTS_DIR=spectra-results-1 npx playwright test --shard=1/2
SPECTRA_RESULTS_DIR=spectra-results-2 npx playwright test --shard=2/2
npx spectra generate -r spectra-results-1 -r spectra-results-2
```

In the Action: `results-dir: "spectra-results-1 spectra-results-2"`.

---

## AI hints

AI is **off unless a key is set**. When it is on, Spectra:

1. groups failures deterministically (no AI) by a normalized error signature: numbers, ids and long strings are masked, and the locator is kept;
2. sends **one** prompt per *new* group, with the signature, up to 3 example errors, code snippets and step titles;
3. stores the answer in `history.json` by signature, so the same failure tomorrow costs nothing;
4. never fails your build because of AI. Errors and timeouts show on the group card instead.

**Never sent:** screenshots, videos, traces, environment variables. **Masked before sending:** bearer tokens, `Authorization` headers, Anthropic, OpenAI, GitHub, AWS and Slack keys, JWTs, passwords in URLs and `key=value`, private keys and emails.

| Env var / Action input | Default | Purpose |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` / `anthropic-api-key` | none | Enables Anthropic |
| `OPENAI_API_KEY` / `openai-api-key` | none | Enables OpenAI or a compatible API |
| `SPECTRA_AI_PROVIDER` / `ai-provider` | auto | `anthropic` or `openai` |
| `SPECTRA_AI_MODEL` / `ai-model` | `claude-haiku-5-5` / `gpt-4.1-mini` | Any model your key can use |
| `SPECTRA_AI_BASE_URL` / `ai-base-url` | provider default | Azure OpenAI, Ollama (`http://localhost:11434/v1`), proxies |
| `SPECTRA_AI_MAX_CLUSTERS` / `ai-max-groups` | `10` | Cap on calls per run |
| `SPECTRA_AI_MAX_CHARS` | `6000` | Cap on prompt size per group |
| `SPECTRA_AI_TIMEOUT_MS` | `30000` | Per-call timeout |
| `SPECTRA_AI=off` | none | Force off |

AI text is always labelled as a hint with a confidence score. Treat it as a starting point, not a diagnosis.

---

## Action reference

| Input | Default | Description |
| --- | --- | --- |
| `results-dir` | `spectra-results` | Reporter output folder(s), space-separated |
| `junit` | none | JUnit XML file(s) to import |
| `report-dir` | `spectra-report` | Where the report is built on the runner |
| `run-name` | none | Name in the report header |
| `publish` | `true` | Push to GitHub Pages |
| `pages-branch` | `gh-pages` | Pages branch |
| `pages-url` | `https://<owner>.github.io/<repo>` | Override for custom domains |
| `keep-runs` | `20` | Run folders kept per branch |
| `comment-on-pr` | `true` | Sticky PR comment |
| `job-summary` | `true` | Summary on the Actions run page |
| `github-token` | `github.token` | Needs `contents: write`, `pull-requests: write` |
| `fail-on-failure` | `false` | Fail the step when tests failed |
| AI inputs | none | See the AI table above |

**Outputs:** `report-url`, `total`, `passed`, `failed`, `flaky`, `pass-rate`.

**Forked PRs** get a read-only token, so Spectra skips publishing and commenting there and still writes the job summary.

The Action runs on Linux runners (`ubuntu-*`) and needs Node 18+, which GitHub runners already have.

---

## Using it with other frameworks (JUnit XML)

TestNG, JUnit 5, pytest, Cucumber-JVM, Jest and most CI tools can emit JUnit XML. Spectra imports it with failure grouping, history and AI hints. It has no step tree or attachments yet.

```yaml
- uses: surkuar/spectra-report@v0
  if: always()
  with:
    results-dir: ''
    junit: target/surefire-reports/TEST-*.xml
```

---

## How it compares

| | Spectra | Allure Report 3 | Playwright HTML reporter |
| --- | --- | --- | --- |
| Failures grouped by root cause | ✅ | Categories (regex rules you write) | ❌ |
| AI cause + fix per group | ✅ optional | ❌ | ❌ |
| New vs recurring failure | ✅ | via history | ❌ |
| PR comment + job summary | ✅ built in | ❌ | ❌ |
| GitHub Pages publishing + history | ✅ one step | DIY | DIY |
| Framework coverage | Playwright + JUnit XML | Very broad | Playwright |

Allure is broader and more mature. Spectra focuses on triage speed for Playwright teams on GitHub.

---

## Develop

```bash
git clone https://github.com/surkuar/spectra-report && cd spectra-report
npm install
npm test                    # build + unit tests
npm run demo -- --mock-ai   # real Playwright demo suite, 6 simulated runs -> demo-report/
```

`dist/` is committed because the GitHub Action runs it directly. Run `npm run build` before committing, and CI checks that it is up to date.

Project layout:

```
src/playwright.ts     Playwright reporter -> spectra-results/run.json
src/importers/junit.ts JUnit XML importer (zero-dependency parser)
src/cluster.ts        deterministic failure grouping
src/ai.ts             AI client (Anthropic / OpenAI-compatible), prompt, cache
src/redact.ts         secret masking
src/history.ts        history, flake rate, new-failure detection
src/summary.ts        PR comment / job summary markdown
src/generate.ts       builds the static report
viewer/index.html     the report UI (vanilla JS, no build step)
action.yml            composite GitHub Action
```

## Roadmap

- **v0.2:** Cucumber JSON and Allure-results importers, flaky quarantine helper, Slack and Teams notifications.
- **v1.0:** branch vs main comparison, in-browser SQL analytics, AI on screenshots, plugin API, frozen schema v1.

## License

MIT © Suresh Babu
