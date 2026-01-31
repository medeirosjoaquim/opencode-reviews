# OpenCode PR Reviews

Automated code reviews for any GitHub PR using [OpenCode](https://opencode.ai) with DeepSeek AI.

## Features

- Review any public GitHub PR by URL
- Configurable review focus (general, security, performance, code-quality, tests)
- Structured output with severity levels and code snippets
- Posts review directly as a PR comment

## Usage

### Manual Trigger

1. Go to **Actions** tab in this repository
2. Select **Manual PR Review** workflow
3. Click **Run workflow**
4. Enter the PR URL (e.g., `https://github.com/owner/repo/pull/123`)
5. Select review focus (optional)
6. Click **Run workflow**

### Review Output Format

The review is posted as a PR comment with:

```markdown
# DeepSeek AI Code Review - PR #123

## Summary
Brief description of what the PR does.

---

### 🔴 CRITICAL `filename:line`

```typescript
const problematicCode = something;
```

**Problem:** Explanation of the issue.

**Fix:** Suggested solution.

---

**Verdict:** APPROVE | REQUEST_CHANGES | COMMENT
```

### Severity Levels

| Icon | Level | Description |
|------|-------|-------------|
| 🔴 | CRITICAL | Security vulnerabilities, data loss, crashes |
| ⚠️ | WARNING | Bugs, logic errors, missing error handling |
| ℹ️ | INFO | Code style, naming, minor improvements |
| 💡 | SUGGESTION | Optional enhancements |

## Setup

### Required Secrets

| Secret | Description |
|--------|-------------|
| `DEEPSEEK_API_KEY` | Your DeepSeek API key |
| `GH_PAT` | GitHub Personal Access Token with `repo` scope (for cross-repo access) |

### Setting up secrets

1. Go to repository **Settings** > **Secrets and variables** > **Actions**
2. Add `DEEPSEEK_API_KEY` with your DeepSeek API key
3. Add `GH_PAT` with a GitHub PAT that has access to the repos you want to review

## Configuration

Edit `.github/workflows/opencode-manual-review.yml` to customize:

- **Model**: Change `deepseek/deepseek-chat` to another supported model
- **Review prompt**: Modify the review instructions in the `Prepare PR context` step
- **Output format**: Adjust the markdown template in the prompt

## How It Works

1. **Parse PR URL** - Extracts owner, repo, and PR number
2. **Checkout** - Clones the target repository
3. **Prepare Context** - Fetches PR metadata and generates diff
4. **Run Review** - Sends diff to DeepSeek via OpenCode CLI
5. **Post Comment** - Extracts review from JSON output and posts to PR

## Limitations

- Only works with public repositories (or repos accessible by your `GH_PAT`)
- Large PRs may take several minutes to review
- DeepSeek API rate limits apply

## License

MIT
