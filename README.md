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
| `model_name` | Model name to use for code review | `sonnet-4.5` | No |
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
