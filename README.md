# Boxtalk Shared Workflows

Centralized GitHub Actions workflows for Boxtalk applications.

## Available Workflows

### Cursor Code Review

Automated code review using Cursor AI for pull requests.

**Location:** `.github/workflows/cursor-code-review.yml`

### Wiki Sync

Generates a GitNexus wiki for a repository on every push to `main` and syncs the
pages into `BoxTalk/boxtalk-kb`.

**Location:** `.github/workflows/wiki-sync.yml`

> ⚠️ **Read [Wiki Sync — the 128 KB prompt cliff](#wiki-sync--the-128-kb-prompt-cliff)
> before onboarding a large repo, or if a run fails with `Error: spawn E2BIG`.**

## Setup Instructions

### 1. Create the Shared Workflows Repository

1. Create a new GitHub repository named `boxtalk-shared-workflows`
2. Push the contents of this directory to the repository:

```bash
cd boxtalk-shared-workflows
git init
git add .
git commit -m "Initial commit: Add reusable code review workflow"
git remote add origin <your-repo-url>
git push -u origin main
```

### 2. Configure Organization Secrets

Set up the `CURSOR_API_KEY` secret at the organization level for easy access across all repositories:

1. Go to your GitHub Organization Settings
2. Navigate to **Secrets and variables** > **Actions**
3. Click **New organization secret**
4. Name: `CURSOR_API_KEY`
5. Value: Your Cursor API key
6. Repository access: Select "All repositories" or specific repositories

Repos using Wiki Sync additionally need `KB_PAT`, and any repo on
`llm_provider: claude` needs `ANTHROPIC_API_KEY` (see
[the cliff caveat](#wiki-sync--the-128-kb-prompt-cliff)). Secrets are resolved
in the **caller's** context and passed through the `secrets:` block — this
shared-workflows repo holds no secrets of its own.

### 3. Enable Workflows in Each Repository

The following repositories already have the caller workflows configured:

- `boxtalk-report-generator`
- `carrier-bill-ingestor`
- `falcon-etl`
- `boxtalk-billing-backend`
- `boxtalk-billing-batch`
- `box-talk-client`

**Important:** Update the workflow files in each repository to use your actual organization name:

Replace `<your-org>` with your GitHub organization name in:
`.github/workflows/code-review.yml`

```yaml
uses: BoxTalk/boxtalk-shared-workflows/.github/workflows/cursor-code-review.yml@main
```

### 4. Customize Prompts with Prompt Injection (Optional)

The workflow uses a **two-layer prompt system**:

1. **Default Prompt (Mandatory)**: Always loaded from `boxtalk-shared-workflows/.github/prompts/code-review-prompt.txt`
   - Contains core review procedures, rules, and commenting guidelines
   - Ensures consistent quality standards across all repositories

2. **Custom Prompt (Optional)**: Injected on top of the default prompt
   - Add repository-specific instructions
   - Focus on domain-specific concerns
   - Customize review priorities

#### To add custom instructions:

Create `.github/prompts/custom-code-review-prompt.txt` in your repository:

```txt
# Focus on these specific areas for this repository:

Database Query Review:
- Check for N+1 queries
- Verify indexes are used
- Ensure proper transaction handling

API Contract:
- Breaking changes must be flagged
- Backwards compatibility is critical
- Validate response formats match OpenAPI spec
```

The workflow will automatically combine: **Default Prompt + Your Custom Prompt**

## Usage

### Basic Usage

The workflow triggers automatically on pull requests. No additional configuration needed.

### Advanced Configuration

You can customize the workflow behavior by passing inputs:

```yaml
jobs:
  code-review:
    permissions:
      contents: read
      pull-requests: write
    uses: BoxTalk/boxtalk-shared-workflows/.github/workflows/cursor-code-review.yml@main
    secrets:
      CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
    with:
      # Use a different model
      model_name: 'claude-opus-4-5-20251101'

      # Change minimum lines threshold
      min_lines_changed: 20

      # Add custom instructions on top of default prompt
      custom_prompt_path: '.github/prompts/custom-review.txt'
```

### Available Inputs

| Input | Description | Default | Required |
|-------|-------------|---------|----------|
| `custom_prompt_path` | Path to custom prompt file (injected on top of default) | `.github/prompts/custom-code-review-prompt.txt` | No |
| `model_name` | Model name to use for code review | `claude-4-sonnet` | No |
| `min_lines_changed` | Minimum lines changed to trigger review | `10` | No |

### Required Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `CURSOR_API_KEY` | Cursor API key for authentication | Yes |

### Required Permissions

The caller workflow must grant these permissions to the job:

| Permission | Level | Purpose |
|------------|-------|---------|
| `contents` | `read` | Access repository code |
| `pull-requests` | `write` | Post review comments on PRs |

**Important:** These permissions must be set in the caller workflow (in each repository), not in the reusable workflow.

## How It Works

1. **Trigger:** Workflow runs on PR open, synchronize, reopen, or ready_for_review
2. **Check:** Skips if PR is draft or has fewer than minimum lines changed
3. **Deduplication:** Skips if the exact commit SHA was already reviewed
4. **Checkout:** Checks out the PR code
5. **Cursor Install:** Installs Cursor CLI
6. **Prompt:** Loads custom prompt (if exists) or default prompt
7. **Review:** Runs Cursor agent to perform code review
8. **Comment:** Posts review comments on the PR

## Benefits

- **Centralized Maintenance:** Update the workflow once, applies to all repositories
- **Consistent Reviews:** Same review standards across all projects
- **Customizable:** Each project can override prompts and settings
- **Easy Adoption:** Simple 10-line workflow file to add to new repos

## Prompt Customization Examples

These custom prompts are **added on top** of the default prompt. The default prompt contains all the core review procedures and rules, so you only need to add **additional** or **specialized** instructions.

### Example 1: Security-Critical Service

Create `.github/prompts/custom-code-review-prompt.txt`:

```
ADDITIONAL SECURITY REQUIREMENTS FOR THIS SERVICE:

This service handles sensitive financial data. Pay EXTRA attention to:

1. Data Encryption:
   - All PII must be encrypted at rest
   - Verify proper use of encryption helpers

2. Access Control:
   - Every API endpoint must have proper authorization checks
   - Flag any direct database access without permission validation

3. Audit Logging:
   - All data modifications must be logged
   - Flag missing audit trail entries

These requirements are IN ADDITION to the standard security checks.
```

### Example 2: Database-Heavy Backend Service

For services with complex database operations:

```
DATABASE-SPECIFIC REVIEW FOCUS:

This repository contains critical database operations. Add these checks:

Priority Areas:
1. N+1 Query Detection - flag any loops with database calls
2. Transaction Boundaries - verify proper transaction handling
3. Index Usage - check that queries use appropriate indexes
4. Connection Pooling - ensure proper connection management

Performance Thresholds:
- Flag queries that scan more than 1000 rows without indexes
- Flag missing pagination on list endpoints

These checks supplement the standard review process.
```

### Example 3: Frontend Application

For UI-heavy applications:

```
FRONTEND-SPECIFIC CONCERNS:

Additional focus areas for this React application:

1. Accessibility:
   - Check for proper ARIA labels
   - Verify keyboard navigation support
   - Flag missing alt text on images

2. Performance:
   - Flag large bundle imports
   - Check for unnecessary re-renders
   - Verify proper use of useMemo/useCallback

3. User Experience:
   - Validate loading states
   - Check error message clarity
   - Verify responsive design considerations
```

## Wiki Sync

Runs `gitnexus analyze` + `gitnexus wiki` on the calling repo, then syncs the
generated pages into `BoxTalk/boxtalk-kb` under `wiki/<kb_name>/`.

### Basic Usage

`.github/workflows/wiki-sync.yml` in the calling repo:

```yaml
name: Wiki Sync

on:
  push:
    branches: [main]

jobs:
  wiki-sync:
    permissions:
      contents: read
    uses: BoxTalk/boxtalk-shared-workflows/.github/workflows/wiki-sync.yml@main
    with:
      repo_dir: my-repo          # must match a REPO_MAP key in sync-wiki.py
      kb_name: my-repo           # folder name in boxtalk-kb/wiki/
    secrets:
      CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
      KB_PAT: ${{ secrets.KB_PAT }}
```

### Available Inputs

| Input | Description | Default | Required |
|-------|-------------|---------|----------|
| `repo_dir` | Directory name of this repo (must match `REPO_MAP` key in `sync-wiki.py`) | — | Yes |
| `kb_name` | Folder name in `boxtalk-kb/wiki/` for this repo | — | Yes |
| `llm_provider` | Wiki LLM provider: `cursor` or `claude`. **See the cliff caveat below.** | `cursor` | No |
| `wiki_timeout_minutes` | Timeout for the wiki generation step | `60` | No |
| `run_migration_ingest` | Run `ingest-migrations.py` + `synthesize-schema-wiki.py` (tenant_manager only) | `false` | No |
| `node_version` | Node.js version for GitNexus | `'22'` | No |

### Required Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `CURSOR_API_KEY` | Cursor API key | Only when `llm_provider: cursor` |
| `ANTHROPIC_API_KEY` | Anthropic API key for the Claude Code CLI | Only when `llm_provider: claude` |
| `KB_PAT` | GitHub PAT with write access to `BoxTalk/boxtalk-kb` | Yes |

The workflow validates this up front and fails in ~2 seconds with a named error
if the key for the selected provider is missing, rather than dying minutes into
indexing.

---

## Wiki Sync — the 128 KB prompt cliff

**Symptom:** the *Generate wiki* step fails after 1–2 seconds with:

```
  GitNexus Wiki Generator

  Error: spawn E2BIG
```

**Fix:** add one line to the caller's `with:` block, and swap the secret.

```yaml
    with:
      repo_dir: my-repo
      kb_name: my-repo
      llm_provider: claude     # <-- the fix
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}   # instead of CURSOR_API_KEY
      KB_PAT: ${{ secrets.KB_PAT }}
```

### Why it happens

GitNexus's Cursor client passes the **entire LLM prompt as a single `argv`
element** (`gitnexus/dist/core/wiki/cursor-client.js` → `args.push(fullPrompt)`).

Linux caps a *single* argv string at `MAX_ARG_STRLEN` = 32 pages =
**131,072 bytes**. This is a **compile-time kernel constant** — it is *not*
governed by `ulimit -s`, `ulimit -a`, or `sysctl ARG_MAX`. Attempts to raise
those limits cannot fix this and have been removed from the workflow.

GitNexus *does* batch oversized grouping prompts, but only above
`GROUPING_TOKEN_BUDGET` = 100,000 tokens (~400,000 bytes). That leaves a **dead
zone** where a prompt is too big for argv but too small to trigger batching:

| grouping prompt size | behaviour | result |
|---|---|---|
| `< 131,072 B` | fits in one argv element | ✅ OK |
| `131,072 B – ~400,000 B` | too big for argv, too small to batch | ❌ **E2BIG** |
| `> ~400,000 B` | `batchedGrouping()` splits into small calls | ✅ OK |

Note this is **not** about repo size in files or graph nodes — finance-backend
has only 8,316 nodes and still hit it. What matters is the byte length of the
file list plus every exported symbol.

`cursor` is the only affected provider. The `claude` / `codex` / `opencode`
clients send the prompt on **stdin**
(`local-cli-client.js` → `child.stdin.end(stdinText)`), and the
`openai` / `azure` / `openrouter` / `custom` clients POST it as JSON. All are
structurally immune, which is why switching providers is the fix.

### ⚠️ Repos closest to the cliff

Measured 2026-09-10 against gitnexus 1.6.11. **If a repo's grouping prompt
crosses 131,072 B it will start failing — apply `llm_provider: claude` in its
caller file.**

| repo | grouping prompt | headroom | status |
|---|---|---|---|
| boxtalk-billing-backend | 479,172 B | *above batching — safe* | cursor |
| **box-talk-client** | **171,277 B** | **over the cap** | ✅ on `claude` |
| **finance-backend** | **164,497 B** | **over the cap** | ✅ on `claude` |
| ⚠️ boxtalk-rate-sim-engine | 85,926 B | **45,146 B** | cursor — closest at risk |
| ⚠️ boxtalk-api | 79,818 B | **51,254 B** | cursor — watch |
| boxtalk-billing-batch | 66,370 B | 64,702 B | cursor |
| tenant_manager | 51,667 B | 79,405 B | cursor |
| boxtalk-conversation-api | 42,886 B | 88,186 B | cursor |
| finance-frontend | 27,899 B | 103,173 B | cursor |
| ratecard-data-loader | 24,104 B | 106,968 B | cursor |
| all others (13 repos) | 1,909 – 12,963 B | > 118,000 B | cursor |

Per-module prompts carry the same exposure under `cursor`: source is truncated
to `DEFAULT_MAX_TOKENS_PER_MODULE` = 30,000 tokens = 120,000 chars, already 92%
of the cap *before* the call graph is appended. A cursor repo that passes today
can be one large module away from the same failure.

### Measuring a repo's grouping prompt

```bash
cd <repo> && gitnexus analyze          # writes .gitnexus/
node -e '
const GN=require("child_process").execSync("npm root -g").toString().trim()+"/gitnexus/dist";
(async()=>{
  const {getStoragePaths}=await import(GN+"/storage/repo-manager.js");
  const q=await import(GN+"/core/wiki/graph-queries.js");
  const p=await import(GN+"/core/wiki/prompts.js");
  await q.initWikiDb(getStoragePaths(process.cwd()).lbugPath);
  const ex=await q.getFilesWithExports(), all=await q.getAllFiles();
  const m=new Map(ex.map(f=>[f.filePath,f]));
  const files=all.map(fp=>m.get(fp)||{filePath:fp,symbols:[]});
  const up=p.fillTemplate(p.GROUPING_USER_PROMPT,{
    FILE_LIST:p.formatFileListForGrouping(files),
    DIRECTORY_TREE:p.formatDirectoryTree(files.map(f=>f.filePath))});
  const b=Buffer.byteLength(p.GROUPING_SYSTEM_PROMPT+"\n\n---\n\n"+up,"utf8");
  console.log(b+" B  ("+(b>131072?"OVER THE CAP — use llm_provider: claude":(131072-b)+" B headroom")+")");
  await q.closeWikiDb();
})()'
```

### What does NOT fix it

- **`ulimit -s unlimited` / `sysctl ARG_MAX`** — raise the *total* argv budget,
  which is not the binding constraint. `MAX_ARG_STRLEN` is per-argument and
  fixed at compile time.
- **`.gitnexusignore`** — evaluated and rejected. For box-talk-client the bulk
  is genuine source (`src/` = 167 KB of the 171 KB); excluding *all* 234 test
  files saves only 16,548 B, still ~23 KB over the cap. You cannot shrink your
  way under without refusing to document the application.

### Timeouts

`claude` throughput is roughly 1.7–2 pages/min. Raise `wiki_timeout_minutes`
for page-heavy repos — box-talk-client generates **130 pages in ~78 minutes**
and would be killed by the 60-minute default:

```yaml
      llm_provider: claude
      wiki_timeout_minutes: 120
```

### Upstream

The real fix belongs in GitNexus: `cursor-client.js` should write the prompt to
stdin like its three sibling clients already do. Nothing is filed upstream as of
gitnexus **1.6.11**. Once fixed and released, `llm_provider` can be dropped and
the affected repos moved back to `cursor`.

---

## Maintenance

### Updating the Workflow

When you update the reusable workflow:

1. Make changes in `boxtalk-shared-workflows/.github/workflows/cursor-code-review.yml`
2. Commit and push to main
3. Changes automatically apply to all repositories using `@main`

### Version Pinning (Optional)

For stability, you can pin to a specific version:

```yaml
uses: <your-org>/boxtalk-shared-workflows/.github/workflows/cursor-code-review.yml@v1.0.0
```

Then tag releases in the shared workflows repository.

## Troubleshooting

### Workflow Not Running

1. Check if `CURSOR_API_KEY` secret is accessible
2. Verify the organization name in the `uses:` field
3. Ensure PR is not in draft mode
4. Check if changes meet minimum line threshold

### Custom Prompt Not Loading

1. Verify file exists at `.github/prompts/code-review-prompt.txt`
2. Check file permissions and encoding (should be UTF-8)
3. Review workflow logs for prompt loading messages

### Review Skipped

Common reasons:
- PR is in draft status
- Changes are too small (< 10 lines by default)
- Commit SHA was already reviewed
- Check workflow logs for skip reason

## Contributing

To add new reusable workflows:

1. Create workflow in `.github/workflows/` with `workflow_call` trigger
2. Document inputs, secrets, and usage in this README
3. Test with a calling repository before rolling out

## License

Internal use for Boxtalk applications.
