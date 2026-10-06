# Codex PR Review

GitHub Action that reviews pull requests with [Codex CLI](https://github.com/openai/codex) and posts:

- a summary review (overview, walkthrough table, findings count)
- inline comments with severity, explanation, and one-click `suggestion` blocks

Works with OpenAI directly or any OpenAI-compatible endpoint.

## Why

Codex has a built-in GitHub review (`@codex review`), but it needs a ChatGPT plan and runs on OpenAI only. [`openai/codex-action`](https://github.com/openai/codex-action) runs Codex in CI, but posting results back to the PR is up to you.

This action fills that gap:

- Bring your own **API key and provider** (OpenAI, OpenRouter, LiteLLM, self-hosted, ...), including `chat` wire API.
- Inline comments with suggestions are posted out of the box, no extra scripting.
- Findings follow a fixed JSON schema, so the output format stays consistent.

## Usage

```yaml
name: codex review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          fetch-depth: 0
          persist-credentials: false

      - uses: mercenarioZ/codex-review@v1
        with:
          api-key: ${{ secrets.CODEX_API_KEY }}
          base-url: ${{ secrets.CODEX_BASE_URL }}
```

Checkout requirements:

- `ref: head.sha` so inline comment line numbers match the PR head.
- `fetch-depth: 0` so the base branch is available for `git diff`.
- `persist-credentials: false` so the token is not readable by the model.

## Inputs

| Name            | Default                     | Description                                      |
| --------------- | --------------------------- | ------------------------------------------------ |
| `api-key`       | —                           | API key for the model provider (required)        |
| `base-url`      | `https://api.openai.com/v1` | OpenAI-compatible base URL, without `/responses` |
| `model`         | Codex default               | Model name                                       |
| `wire-api`      | `responses`                 | `responses` or `chat`                            |
| `codex-version` | `0.160.1`                   | `@openai/codex` npm version                      |
| `github-token`  | `github.token`              | Token used to post the review                    |

## Project context

Codex reads `AGENTS.md` from the repository root, so put project conventions there to make reviews project-aware.

## Notes

- Runs on Linux runners only. The action relaxes `kernel.apparmor_restrict_unprivileged_userns` on the ephemeral runner so the Codex `read-only` sandbox can start.
- If a finding points at a line outside the diff, GitHub rejects the inline review and the action falls back to posting the summary as a comment.
- Pull requests from forks do not receive secrets; the `if:` above skips them.

## License

MIT
