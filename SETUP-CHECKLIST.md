# Setup Checklist

## Step 1: Create Shared Workflows Repository

- [ ] Create new GitHub repository: `boxtalk-shared-workflows`
- [ ] Push this directory to the repository
- [ ] Verify repository is accessible to your organization

```bash
cd boxtalk-shared-workflows
git init
git add .
git commit -m "Initial commit: Add reusable code review workflow"
git remote add origin git@github.com:<your-org>/boxtalk-shared-workflows.git
git push -u origin main
```

## Step 2: Configure Organization Secret

- [ ] Go to Organization Settings > Secrets and variables > Actions
- [ ] Create new secret: `CURSOR_API_KEY`
- [ ] Set value to your Cursor API key
- [ ] Grant access to all repositories (or specific ones)

## Step 3: Update Repository Workflows

Update the following line in each repository's `.github/workflows/code-review.yml`:

**Replace:** `<your-org>/boxtalk-shared-workflows`
**With:** `YOUR_ORG_NAME/boxtalk-shared-workflows`

### Repositories to Update:

- [ ] boxtalk-report-generator
- [ ] carrier-bill-ingestor
- [ ] falcon-etl
- [ ] boxtalk-billing-backend
- [ ] boxtalk-billing-batch
- [ ] box-talk-client

## Step 4: Optional - Customize Prompts

For each repository that needs custom review criteria:

- [ ] Copy `.github/prompts/code-review-prompt.txt` from shared workflows repo
- [ ] Place in repository at `.github/prompts/code-review-prompt.txt`
- [ ] Customize for project-specific needs

## Step 5: Test

- [ ] Create a test PR in one repository
- [ ] Verify workflow triggers
- [ ] Check that review comments appear
- [ ] Confirm prompt is being used (default or custom)

## Verification Commands

### Check if workflow exists:
```bash
cd <repository-name>
cat .github/workflows/code-review.yml
```

### Check if secret is available (run in GitHub Actions):
```bash
echo "Secret exists: ${{ secrets.CURSOR_API_KEY != '' }}"
```

### Test workflow locally:
```bash
gh workflow list
gh workflow run code-review.yml
```

## Quick Reference

### Workflow Location (Reusable)
`boxtalk-shared-workflows/.github/workflows/cursor-code-review.yml`

### Workflow Location (Caller)
`<repository>/.github/workflows/code-review.yml`

### Default Prompt Location
`boxtalk-shared-workflows/.github/prompts/code-review-prompt.txt`

### Custom Prompt Location (Optional)
`<repository>/.github/prompts/code-review-prompt.txt`

## Need Help?

- Check workflow runs: Go to repository > Actions tab
- View logs: Click on a workflow run > code-review job
- Test secret: Add a step to echo (masked) secret existence
