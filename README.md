# Bitbucket Code Review Plugin for Claude Code

Automated code review for Bitbucket Cloud pull requests using multiple specialized agents with confidence-based scoring.

Forked from [Anthropic's official code-review plugin](https://github.com/anthropics/claude-code-plugins) and adapted for Bitbucket Cloud REST API.

## Installation

```bash
/plugin install bitbucket-code-review@yoonjong12
```

Or install from local path:
```bash
/plugin install /path/to/bitbucket-code-review
```

## Prerequisites

Set these environment variables (in `.env` or shell):

```bash
BITBUCKET_EMAIL="your@email.com"
BITBUCKET_API_TOKEN="ATATTxxx..."
```

Create an app password at: https://bitbucket.org/account/settings/app-passwords/
Required scopes: `pullrequest:read`, `pullrequest:write`

## Usage

```bash
/bitbucket-code-review:code-review
```

The plugin will:
1. Check PR eligibility (not closed, not already reviewed)
2. Collect CLAUDE.md files for compliance checking
3. Launch 5 parallel review agents (CLAUDE.md audit, bug scan, git history, prior PR comments, code comments)
4. Score each finding (0-100 confidence)
5. Post filtered results (score >= 80) as a PR comment

## Differences from GitHub version

| Feature | GitHub (original) | Bitbucket (this fork) |
|---------|-------------------|----------------------|
| CLI tool | `gh` | `curl` with REST API v2.0 |
| Auth | `GITHUB_TOKEN` | Basic auth (`BITBUCKET_EMAIL:BITBUCKET_API_TOKEN`) |
| Code links | `github.com/.../blob/{sha}/file#L10-L15` | `bitbucket.org/.../src/{sha}/file#lines-10:15` |
| PR interaction | `gh pr view/comment` | Bitbucket REST API endpoints |

## License

MIT
