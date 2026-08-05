# Profile automation

These workflows live in the special profile repository `erfzrezr/erfzrezr`. They only
maintain the presentation of the profile README. None of them build or test code.

Every workflow declares an explicit least-privilege `permissions:` block and a
`concurrency:` group, and every third-party action is pinned to a version tag.

| Workflow | Trigger | Needs | Writes | State |
|---|---|---|---|---|
| `update-readme.yml` | daily cron `23 4 * * *`, manual | built-in `GITHUB_TOKEN` | `README.md` | active: `README.md` ships both `LAST_UPDATED` markers, so the first run stamps a date |
| `metrics.yml` | manual only, commented-out weekly cron `17 5 * * 0` | `METRICS_TOKEN` secret, plus built-in `GITHUB_TOKEN` to commit | `github-metrics.svg` | gated off: needs the secret, and needs an account with something to summarise. The README's metrics `<img>` is commented out as well, so switching it on is a two-part change |
| `snake.yml` | manual only, commented-out cron `0 */12 * * *` | built-in `GITHUB_TOKEN` | `output` branch | gated off until the contribution graph has density |

## update-readme.yml

Stamps a date into the profile README between two marker comments. Self-contained:
`actions/checkout` plus one shell step. No third-party action, no secret.

The README must contain both markers, each alone on its own line:

```
<!-- LAST_UPDATED:START -->
Last updated: 2026-01-01
<!-- LAST_UPDATED:END -->
```

The profile `README.md` in this build ships both markers, each exactly once and each on
its own line, so the first run stamps a date. The guards below cover the case where a
marker is later removed, duplicated, or reordered by an edit to the README: the job
then logs a warning, changes nothing, and exits green.

Behaviour:

- The job logs a warning and exits successfully without modifying the file when
  `README.md` is absent, when either marker is missing, when both markers share one
  line, when either marker appears more than once, or when the end marker precedes
  the start marker. Each of those cases would otherwise let the rewrite delete real
  content, so the job refuses instead of guessing.
- The stamp has day granularity, so a second run on the same UTC day produces a
  byte-identical file. The job checks `git diff --quiet` and skips the commit. It
  never creates an empty commit.
- The rewritten file is checked for both markers before it replaces `README.md`. If
  either is missing the original is left untouched and the job fails loudly.
- It pushes normally. No force push, no rebase over anyone else's commit. If a
  human pushed in the same instant, the push fails and the next run stamps it.
- `permissions:` is `contents: write` and nothing else, which is the minimum needed
  to commit one file back to this repository.

## metrics.yml: gated off on purpose

Renders `github-metrics.svg` with `lowlighter/metrics`.

The `base` set is `header, activity, community, repositories, metadata`, all of which
are drawn from public repositories and contribution activity. The account has neither
yet, so the image would be close to empty. The workflow therefore ships with
`workflow_dispatch:` only and a commented-out weekly `schedule:` (`17 5 * * 0`), the
same gate `snake.yml` uses. Uncomment the `schedule:` block once there is a public
repository and a few months of commits. A manual run is safe at any time and is the
way to preview the output first.

Enabling metrics is a two-part change. The workflow trigger is one part. The other is
the profile README, which carries the metrics `<img>` inside an HTML comment: the URL
404s until the workflow has committed `github-metrics.svg` at least once, so the image
stays commented out until then.

**Setup is required before the first successful run.** `lowlighter/metrics` reads
account-level data, which the built-in `GITHUB_TOKEN` cannot do, so it needs a
classic personal access token stored as the repository secret `METRICS_TOKEN`.

1. Create the token: <https://github.com/settings/tokens/new?scopes=read:user&description=METRICS_TOKEN>
   - `read:user` — required.
   - `repo` — optional, and only if private repositories should be counted in the
     statistics. It grants full read/write on every repository you own, so leave it
     off unless you need private stats.
2. Add the secret: <https://github.com/erfzrezr/erfzrezr/settings/secrets/actions/new>
   - Name `METRICS_TOKEN`, value the token string.
3. Run it once by hand: Actions tab, `metrics`, "Run workflow".
4. Uncomment the metrics `<img>` line in the profile README once the output is worth
   showing. The same markup is in the header comment of `metrics.yml`.

The workflow commits the SVG using the workflow's own `GITHUB_TOKEN` via
`committer_token`, so `METRICS_TOKEN` never needs write scope.

Until the secret is set, the run fails at the metrics step. Nothing else depends on
it. Rotate the token before it expires; the workflow fails visibly rather than
serving a stale image.

## snake.yml: disabled on purpose

`Platane/snk` animates a snake eating the contribution graph. The account currently
has no contribution history, so the animation would render as an empty grid. That is
a worse first impression than no animation.

The workflow is therefore committed with `workflow_dispatch:` only and a
commented-out `schedule:` block that would otherwise run every 12 hours
(`0 */12 * * *`).

Enable it after roughly three months of steady commits, when
<https://github.com/erfzrezr> shows a visibly populated contribution graph:

1. Uncomment the `schedule:` block in `snake.yml`.
2. Push, then trigger one run by hand from the Actions tab.
3. Embed the generated images in the README. They are published to the `output`
   branch, not to `main`:

```html
<picture>
  <source media="(prefers-color-scheme: dark)"
    srcset="https://raw.githubusercontent.com/erfzrezr/erfzrezr/output/github-snake-dark.svg">
  <img alt="contribution snake"
    src="https://raw.githubusercontent.com/erfzrezr/erfzrezr/output/github-snake.svg">
</picture>
```

## Other things intentionally left out

- **Streak card, trophy card, and activity graph.** All three render as zeroes on a
  new account. They are present in the README as HTML comments, to be uncommented
  later.
- **Visitor-counter badges.** `komarev.com/ghpvc` and `visitor-badge.laobi.icu` both
  failed availability checks. A broken image on the first line of a profile is worse
  than no counter.
- **Scheduled dependency or code scanning.** There is no code in the profile
  repository to scan. Those workflows belong in the project repositories.
