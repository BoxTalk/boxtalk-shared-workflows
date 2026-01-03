# Boxtalk Shared Workflows

Centralized GitHub Actions workflows for Boxtalk applications.

## Available Workflows

### Cursor Code Review

Automated code review using Cursor AI for pull requests.

**Location:** `.github/workflows/cursor-code-review.yml`

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

### 4. Customize Prompts (Optional)

Each repository can use a custom prompt by creating:

```
.github/prompts/code-review-prompt.txt
```

If this file exists, the workflow will use it. Otherwise, it falls back to the default prompt in the shared workflows repository.

#### To customize a prompt:

1. Copy the default prompt from `boxtalk-shared-workflows/.github/prompts/code-review-prompt.txt`
2. Create `.github/prompts/code-review-prompt.txt` in your application repository
3. Modify it according to your project's needs

## Usage

### Basic Usage

The workflow triggers automatically on pull requests. No additional configuration needed.

### Advanced Configuration

You can customize the workflow behavior by passing inputs:

```yaml
jobs:
  code-review:
    uses: BoxTalk/boxtalk-shared-workflows/.github/workflows/cursor-code-review.yml@main
    secrets:
      CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
    with:
      # Use a different model
      model_name: 'claude-opus-4-5-20251101'

      # Change minimum lines threshold
      min_lines_changed: 20

      # Use a custom prompt path
      prompt_path: '.github/prompts/custom-review.txt'
```

### Available Inputs

| Input | Description | Default | Required |
|-------|-------------|---------|----------|
| `prompt_path` | Path to the code review prompt file | `.github/prompts/code-review-prompt.txt` | No |
| `model_name` | Model name to use for code review | `claude-sonnet-4-5-20250929` | No |
| `min_lines_changed` | Minimum lines changed to trigger review | `10` | No |

### Required Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `CURSOR_API_KEY` | Cursor API key for authentication | Yes |

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

### Example 1: Stricter Security Focus

Create `.github/prompts/code-review-prompt.txt` in your repo:

```
You are operating in a GitHub Actions runner performing automated code review with STRICT SECURITY FOCUS.

Context:
  - Repo: {{REPO}}
  - PR Number: {{PR_NUMBER}}

Review Procedure:
1. Get current diff: gh pr diff {{PR_NUMBER}}
2. Focus EXCLUSIVELY on:
   - SQL injection vulnerabilities
   - Authentication/authorization bypasses
   - Sensitive data exposure
   - Dependency vulnerabilities

Commenting Rules:
- ONLY comment on security issues (🔒)
- Max 3 comments per review
- Must be actionable with specific fix recommendations
```

### Example 2: Backend-Specific Review

For backend services, focus on API design, database queries, and error handling:

```
Context: Backend service review for {{REPO}}

Focus Areas:
1. API Design (RESTful conventions, status codes)
2. Database Query Performance (N+1, missing indexes)
3. Error Handling (logging, user-facing messages)
4. Input Validation
5. Transaction Management

Ignore: Frontend concerns, styling, formatting
```

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
