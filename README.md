# TeamNickHart

Org-level defaults and reusable GitHub Actions workflows, shared across this
organization's repositories.

This repo holds no application code. It exists so that a change to a check is
made once rather than three times — and because a workflow that fails to load
reports only `startup_failure`, with no logs, which is much less painful to
debug in one place than in every repo that copied it.

## Reusable workflows

| Workflow | Does | Blocking? |
|---|---|---|
| [`mdx-check.yml`](.github/workflows/mdx-check.yml) | Compiles every post with the site's own contentlayer config, so a post that would break the build cannot merge | **Yes** — meant to be a required check |
| [`prose-check.yml`](.github/workflows/prose-check.yml) | Spelling (`cspell`) and markdown style (`markdownlint`), reported to the job summary | No — advisory by design |
| [`notify-author.yml`](.github/workflows/notify-author.yml) | Emails a post's author a link to their Vercel preview and its pull request | No — a missing notification is a courtesy not delivered, not a reason to fail a build |

### mdx-check

```yaml
name: MDX check
on:
  pull_request:

jobs:
  mdx:
    permissions:
      contents: read
    uses: TeamNickHart/.github/.github/workflows/mdx-check.yml@v1
```

| Input | Default | |
|---|---|---|
| `node-version` | `'24'` | Match what the site builds with, or CI and production can disagree |

Runs the site's own `contentlayer2 build` rather than a plugin list maintained
here, because a copy of that list would drift from what the sites actually build
with — and a check that disagrees with the real build is worse than no check, in
both directions.

**Do not add a `paths:` filter.** A required check that never runs never
reports, leaving the pull request blocked with nothing to click and no way to
merge. This is not hypothetical: a README-only pull request hit exactly that,
with every visible check green and no way forward.

### prose-check

```yaml
name: Prose check
on:
  pull_request:

jobs:
  prose:
    permissions:
      contents: read
    uses: TeamNickHart/.github/.github/workflows/prose-check.yml@v1
```

| Input | Default | |
|---|---|---|
| `node-version` | `'24'` | |
| `content-glob` | `'data/**/*.mdx'` | Files to check, relative to the repo root |

Tune it with two committed files in the calling repo: `cspell.json` for words
that are correct but not in a dictionary, and `.markdownlint-cli2.jsonc` for
rules that misread MDX.

**Do not make this a required check.** It is built to succeed even when it finds
something, so requiring it would be meaningless — and making it fail would block
a merge on a spelling opinion. Findings go to the job summary, which needs no
permissions at all, where a pull request comment would need
`pull-requests: write` — a real escalation on a token that could then modify
pull requests.

### notify-author

```yaml
name: Notify author
on:
  deployment_status:

jobs:
  notify:
    permissions:
      contents: read
      pull-requests: read
    uses: TeamNickHart/.github/.github/workflows/notify-author.yml@v1
    with:
      mail-from: ${{ vars.NOTIFY_FROM }}
      shared-ref: v1
    secrets:
      RESEND_API_KEY: ${{ secrets.RESEND_API_KEY }}
      AUTHOR_EMAIL_MAP: ${{ secrets.AUTHOR_EMAIL_MAP }}
```

| Input | Default | |
|---|---|---|
| `mail-from` | **required** | Sender address. A called workflow cannot read the caller's `vars`, so this is passed in |
| `shared-ref` | `main` | See [Versioning](#versioning) |
| `content-path` | `data/blog` | |
| `route-prefix` | `/blog` | |
| `branch-prefix` | `post-inbox/` | Only branches with this prefix notify |

| Secret | |
|---|---|
| `RESEND_API_KEY` | Resend API key. One key can serve several sites |
| `AUTHOR_EMAIL_MAP` | JSON mapping an author name to an address. Never committed — the post's frontmatter carries only the name |

No address is ever committed or logged: it is looked up from the secret at send
time and masked in output, because an Actions log outlives the run and is
readable by anyone with repo access.

## Two things that will cost you an afternoon

**`permissions` must sit on the *calling job*, not at the top level of the
caller's file.** A top-level block does not reach a `workflow_call` job — the
token silently falls back to the repo default, and `notify-author`'s pull
request lookup returns 403, so the email links the pull request *list* instead
of the pull request. The `GITHUB_TOKEN Permissions` group at the top of a run's
log shows what the job actually got; that is the first thing to read when a
lookup 403s.

Passing the caller's `GITHUB_TOKEN` down as a secret does **not** work around a
missing grant. It is the same token with the same permissions — and GitHub
rejects `secrets.GITHUB_TOKEN` passed by name into a reusable workflow, which
fails the run outright.

**A called workflow cannot discover its own version.** `github.job_workflow_sha`
does not exist; `github.workflow_sha` and `github.workflow_ref` both describe
the *caller*. So `notify-author` takes `shared-ref`, and the caller passes the
same ref it pinned the workflow to. Omit it and the script comes from `main`,
which means a caller pinned to `@v1` runs whatever script is newest.

## Versioning

Callers pin `@v1`. The tag moves when a change is ready for every site, so a fix
reaches all of them by moving one tag rather than one commit per repo.

`v1` is a **mutable** tag: a release boundary, not an immutability guarantee.
Anyone who can push here can retarget it. If you need the stronger property, pin
a caller to a commit SHA instead — `v1` then still serves as a label for which
commit that is.

## Contributing

CI runs on every pull request:

- **`actionlint`**, which validates expressions against the real context types
  and runs `shellcheck` over every `run:` block
- **`node --check`** on the shared script, which is fetched and executed at
  runtime — so a syntax error there would surface as a failed notification
- a check that every `workflow_call` input and secret in use is declared, since
  an undeclared one parses fine and arrives empty

Locally:

```bash
brew install actionlint
actionlint
node --check .github/scripts/notify-author.mjs
```

**What CI cannot catch:** whether a context is actually *populated* inside a
called workflow. `vars.X` is unavailable there and `github.job_workflow_sha`
does not exist at all — both lint clean and arrive empty at runtime. Print a
value before depending on it.

## Projects

| | |
|---|---|
| [**md2do**](https://github.com/TeamNickHart/md2do) | Track and manage TODOs in markdown, with MCP, VS Code, Obsidian and Todoist integrations. [md2do.com](https://md2do.com) · [`@md2do/cli`](https://www.npmjs.com/package/@md2do/cli) |
| [**post-inbox**](https://github.com/TeamNickHart/post-inbox) | Email a post to a Git-based blog. A Cloudflare Worker turns an email into a draft pull request with a live preview, and never publishes directly |

## License

MIT. See [LICENSE](LICENSE).
