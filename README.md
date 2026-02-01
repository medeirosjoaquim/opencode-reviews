# OpenCode PR Reviews

Automated AI code reviews for any GitHub PR using [OpenCode](https://opencode.ai) with DeepSeek.

## Features

- Review any GitHub PR by URL
- Configurable review focus (general, security, performance, code-quality, tests)
- Adjustable strictness levels (critical, medium, low)
- Structured output with severity levels and code snippets
- Posts review directly as a PR comment

## Quick Start

### 1. Create the workflow file

Create `.github/workflows/opencode-review.yml` in your repository:

```yaml
name: AI PR Review

on:
  workflow_dispatch:
    inputs:
      pr_url:
        description: 'PR URL (e.g., https://github.com/owner/repo/pull/123)'
        required: true
        type: string
      review_focus:
        description: 'Review focus'
        required: false
        type: choice
        default: 'general'
        options:
          - general
          - security
          - performance
          - code-quality
          - tests
      strictness:
        description: 'critical=fast/security only, medium=standard, low=thorough/nitpicks'
        required: false
        type: choice
        default: 'medium'
        options:
          - critical
          - medium
          - low

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - name: Parse PR URL
        id: parse
        run: |
          PR_URL="${{ inputs.pr_url }}"
          if [[ $PR_URL =~ github\.com/([^/]+)/([^/]+)/pull/([0-9]+) ]]; then
            echo "owner=${BASH_REMATCH[1]}" >> $GITHUB_OUTPUT
            echo "repo=${BASH_REMATCH[2]}" >> $GITHUB_OUTPUT
            echo "pr_number=${BASH_REMATCH[3]}" >> $GITHUB_OUTPUT
            echo "repo_full=${BASH_REMATCH[1]}/${BASH_REMATCH[2]}" >> $GITHUB_OUTPUT
          else
            echo "Invalid PR URL format" && exit 1
          fi

      - name: Checkout target repository
        uses: actions/checkout@v4
        with:
          repository: ${{ steps.parse.outputs.repo_full }}
          token: ${{ secrets.GH_PAT }}
          fetch-depth: 0

      - name: Prepare PR context
        id: pr_info
        run: |
          REPO_FULL="${{ steps.parse.outputs.repo_full }}"
          PR_NUMBER="${{ steps.parse.outputs.pr_number }}"

          PR_DATA=$(gh pr view $PR_NUMBER -R $REPO_FULL --json headRefName,baseRefName,title,author)
          HEAD_REF=$(echo "$PR_DATA" | jq -r '.headRefName')
          BASE_REF=$(echo "$PR_DATA" | jq -r '.baseRefName')
          PR_TITLE=$(echo "$PR_DATA" | jq -r '.title')
          PR_AUTHOR=$(echo "$PR_DATA" | jq -r '.author.login')

          git fetch origin $HEAD_REF $BASE_REF
          echo "head_ref=$HEAD_REF" >> $GITHUB_OUTPUT
          echo "base_ref=$BASE_REF" >> $GITHUB_OUTPUT

          cat > review_prompt.md << 'EOF'
          You are a senior code reviewer. Review this pull request.
          EOF

          cat >> review_prompt.md << EOF
          ## PR Information
          - Repository: $REPO_FULL
          - PR: #$PR_NUMBER - $PR_TITLE
          - Author: @$PR_AUTHOR
          - Focus: ${{ inputs.review_focus }}
          - Strictness: ${{ inputs.strictness }}

          ## Diff
          EOF

          echo '```diff' >> review_prompt.md
          git diff -U5 origin/$BASE_REF...origin/$HEAD_REF >> review_prompt.md
          echo '```' >> review_prompt.md

          cat >> review_prompt.md << 'EOF'

          ## Instructions

          **Strictness levels (affects both speed and detail):**
          - critical: FAST review - only scan the diff for security vulnerabilities and obvious bugs. Skip codebase exploration, skip style issues. Output only critical findings.
          - medium: Standard review - check for bugs, missing error handling, and code quality issues
          - low: Thorough review - include style nitpicks, naming suggestions, and minor improvements

          IMPORTANT: Do NOT use todo lists or explore files outside the diff. Analyze the diff directly and output findings immediately.

          Output GitHub markdown. For each issue use EXACTLY this format:

          ---
          ### 🔴 CRITICAL `filename:line-line`

          **Original Code:**
          ```language
          XX |   problematic code here
          ```

          **Problem:** Explanation of the issue.

          **Suggested Fix:**
          ```language
          XX |   fixed code here
          ```
          ---

          IMPORTANT: Always include "**Original Code:**" and "**Suggested Fix:**" labels before each code block.

          Severities: 🔴 CRITICAL, ⚠️ WARNING, ℹ️ INFO, 💡 SUGGESTION
          End with: **Verdict:** APPROVE | REQUEST_CHANGES | COMMENT
          EOF
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}

      - name: Install OpenCode
        run: curl -fsSL https://opencode.ai/install | bash

      - name: Run AI Review
        run: |
          export PATH="$HOME/.opencode/bin:$PATH"
          opencode run "$(cat review_prompt.md)" --model deepseek/deepseek-chat --format json 2>&1 | tee review_raw.json || true

          grep '"type":"text"' review_raw.json | while read -r line; do
            text=$(echo "$line" | jq -r '.part.text // empty')
            if echo "$text" | grep -qi "verdict\|summary\|problem"; then
              echo "$text" > review_output.md
              break
            fi
          done

          [ -s review_output.md ] || grep '"type":"text"' review_raw.json | tail -1 | jq -r '.part.text // empty' > review_output.md
          [ -s review_output.md ] || echo "Review generation failed" > review_output.md
        env:
          DEEPSEEK_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}

      - name: Post Review Comment
        run: |
          gh pr comment ${{ steps.parse.outputs.pr_number }} \
            -R ${{ steps.parse.outputs.repo_full }} \
            --body-file review_output.md
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
```

### 2. Add repository secrets

Go to your repository **Settings** → **Secrets and variables** → **Actions** and add:

| Secret | Description | How to get it |
|--------|-------------|---------------|
| `DEEPSEEK_API_KEY` | DeepSeek API key | [platform.deepseek.com](https://platform.deepseek.com/) |
| `GH_PAT` | GitHub Personal Access Token | See [Creating a PAT](#creating-a-github-pat) |

### 3. Run a review

1. Go to **Actions** tab
2. Select **AI PR Review**
3. Click **Run workflow**
4. Enter a PR URL and click **Run workflow**

---

## Creating a GitHub PAT

1. Go to [GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)](https://github.com/settings/tokens)
2. Click **Generate new token (classic)**
3. Give it a name (e.g., "OpenCode PR Reviews")
4. Select scopes:
   - `repo` (Full control of private repositories)
   - `write:discussion` (optional, for PR comments)
5. Click **Generate token**
6. Copy the token and add it as `GH_PAT` secret

> **Note:** For reviewing PRs in other repositories, your PAT needs access to those repos.

---

## Review Output Format

Reviews are posted as PR comments with structured markdown:

```markdown
# AI Code Review - PR #123

## Summary
Brief description of what the PR does.

---

### 🔴 CRITICAL `src/auth.ts:45-47`

**Original Code:**
```typescript
45 |   const password = req.body.password;
46 |   await db.users.insert({ email, password });
47 |   return res.json({ success: true });
```

**Problem:** Password is stored without hashing, exposing user credentials if database is compromised.

**Suggested Fix:**
```typescript
45 |   const password = req.body.password;
46 |   const hashedPassword = await bcrypt.hash(password, 10);
47 |   await db.users.insert({ email, password: hashedPassword });
48 |   return res.json({ success: true });
```

---

### ⚠️ WARNING `src/api.ts:120-121`

**Original Code:**
```typescript
120 |   const data = await fetch(url);
121 |   return data.json();
```

**Problem:** Missing error handling for network request could cause unhandled exceptions.

**Suggested Fix:**
```typescript
120 |   try {
121 |     const data = await fetch(url);
122 |     return data.json();
123 |   } catch (error) {
124 |     console.error('Fetch failed:', error);
125 |     throw new ApiError('Network request failed');
126 |   }
```

---

**Verdict:** REQUEST_CHANGES
```

### Severity Levels

| Icon | Level | Description |
|------|-------|-------------|
| 🔴 | CRITICAL | Security vulnerabilities, data loss, crashes |
| ⚠️ | WARNING | Bugs, logic errors, missing error handling |
| ℹ️ | INFO | Code style, naming, minor improvements |
| 💡 | SUGGESTION | Optional enhancements |

### Strictness Levels

Control review speed and thoroughness:

| Level | Speed | What gets reported |
|-------|-------|-------------------|
| **critical** | Fast | Security vulnerabilities, crashes, data loss - skips codebase exploration |
| **medium** | Standard | Above + code quality, error handling, potential bugs |
| **low** | Thorough | Above + style nitpicks, naming, minor improvements |

---

## Configuration

### Using a different AI model

Change the `--model` parameter in the workflow:

```yaml
opencode run "$(cat review_prompt.md)" --model deepseek/deepseek-reasoner --format json
```

Available models depend on your OpenCode configuration and API keys.

### Customizing the review prompt

Edit the `review_prompt.md` section in the workflow to change:
- Review focus areas
- Output format
- Severity definitions
- Language/framework-specific instructions

### Auto-trigger on PR events

Add this to the `on:` section to automatically review new PRs:

```yaml
on:
  pull_request:
    types: [opened, synchronize]
  workflow_dispatch:
    # ... keep existing inputs
```

Then modify the workflow to use `${{ github.event.pull_request.number }}` instead of the input URL.

---

## Limitations

- Large PRs (1000+ lines) may take several minutes
- DeepSeek API rate limits apply
- Requires PAT with access to target repositories
- Review quality depends on the AI model used

---

## Troubleshooting

### "Resource not accessible by integration"

Your `GH_PAT` doesn't have access to the target repository. Ensure the token has `repo` scope and you have access to the repository.

### Empty review output

Check the workflow logs for the "Run AI Review" step. Common issues:
- Invalid `DEEPSEEK_API_KEY`
- Large PR exceeding context limits
- Network timeout

### Review not posted

Ensure `GH_PAT` has write access to the target repository's pull requests.

---

## License

MIT
