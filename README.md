# Codex PR Review

GitHub Action that reviews pull requests with [Codex CLI](https://github.com/openai/codex) and posts:

- a summary review (overview, walkthrough table, findings count)
- inline comments with severity, explanation, and one-click `suggestion` blocks
- a ready-to-paste `Prompt for AI` block to hand the fixes to a coding agent
- incremental reviews: only the commits pushed since the last review are sent to the model

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
    types: [opened, ready_for_review, synchronize, reopened]

concurrency:
  group: codex-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: write # required to resolve review threads; use read + resolve-fixed: false instead
  pull-requests: write

jobs:
  review:
    if: >-
      github.event.pull_request.draft == false &&
      github.event.pull_request.head.repo.full_name == github.repository
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

Least privilege, without thread resolution (`contents: read` alone cannot resolve threads):

```yaml
permissions:
  contents: read
  pull-requests: write

# ... same job as above, with:
      - uses: mercenarioZ/codex-review@v1
        with:
          api-key: ${{ secrets.CODEX_API_KEY }}
          base-url: ${{ secrets.CODEX_BASE_URL }}
          resolve-fixed: false
```

The workflow token is never written into the git config (`persist-credentials: false`) and is passed
only to the steps that call the API, so the model never sees it.

## Inputs

| Name            | Default                     | Description                                      |
| --------------- | --------------------------- | ------------------------------------------------ |
| `api-key`       | —                           | API key for the model provider (required)        |
| `base-url`      | `https://api.openai.com/v1` | OpenAI-compatible base URL, without `/responses` |
| `model`         | Codex default               | Model name                                       |
| `wire-api`      | `responses`                 | `responses` or `chat`                            |
| `codex-version` | `0.160.1`                   | `@openai/codex` npm version                      |
| `incremental`   | `true`                      | Review only commits pushed since the last review |
| `resolve-fixed` | `true`                      | Resolve threads whose finding is fixed (needs `contents: write`) |
| `github-token`  | `github.token`              | Token used to post the review                    |

## Outputs

| Name        | Description                                                                     |
| ----------- | ------------------------------------------------------------------------------- |
| `ai-prompt` | `Prompt for AI` block of the review, empty when nothing was reviewed            |

Use it to hand the fixes straight to another agent:

```yaml
      - uses: mercenarioZ/codex-review@v1
        id: review
        with:
          api-key: ${{ secrets.CODEX_API_KEY }}

      - if: steps.review.outputs.ai-prompt != ''
        run: echo "${{ steps.review.outputs.ai-prompt }}" | your-coding-agent
```

## Incremental reviews

The action remembers the last reviewed commit in a hidden marker
(`<!-- codex-review:head=<sha> -->`) inside the review body, and diffs from that commit on the next
run. It falls back to the full PR diff when:

- the PR has no review from this action yet
- the marked commit is gone from the fetched history (force-push)
- the marked commit is not an ancestor of the head (amend, rebase, force-push)
- `incremental: false`

A skipped run (same head, or an empty range) exits green without calling the model.

## Auto-resolving fixed findings

On every run the action lists the review threads it opened before, hands the still-open ones to the
model, and asks which of them are already fixed in the current code. Fixed threads are resolved
through the GraphQL `resolveReviewThread` mutation; the rest stay open. The model only resolves a
thread when the current code no longer has the problem, so a wrong resolution cannot hide a real bug
without also being reported as a new finding.

- Only threads whose root comment carries the hidden `<!-- codex-review:comment -->` marker are
  managed, so human threads and comments from older versions are never touched.
- Resolving needs `contents: write` in the workflow permissions, as in the [Usage](#usage) example.
  With `contents: read` the mutation returns `Resource not accessible by integration`; the action logs
  a warning and keeps the rest of the run green. See the "Least privilege" variant right below Usage
  (`contents: read` + `resolve-fixed: false`) when you do not want to grant write.
- Thread ids the model returns are validated against the open list before any mutation is sent.

## Project context

Codex reads `AGENTS.md` from the repository root, so put project conventions there to make reviews project-aware.

## Notes

- Runs on Linux runners only. The action relaxes `kernel.apparmor_restrict_unprivileged_userns` on the ephemeral runner so the Codex `read-only` sandbox can start.
- If a finding points at a line outside the diff, GitHub rejects the inline review and the action falls back to posting the summary as a comment.
- Pull requests from forks do not receive secrets; the `if:` above skips them.

## License

MIT
