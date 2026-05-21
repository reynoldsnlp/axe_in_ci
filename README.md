# axe-in-ci

Proof of concept: run [axe-cli](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/cli)
accessibility scans against a static site in GitHub Actions.

## How it works

The `Accessibility (axe-cli)` workflow (`.github/workflows/a11y.yml`):

1. Checks out the repo and installs Node 20.
2. Installs `@axe-core/cli` globally.
3. Starts `python3 -m http.server 8000` in the background from `site/`, then
   polls `http://localhost:8000/` until it answers.
4. Runs `axe` against the URLs to scan. `--exit` causes axe-cli to exit
   non-zero when violations are found, failing the CI job.
5. Uploads the JSON report as a build artifact (`axe-report`).

axe-cli ships with a headless Chromium driver, so no extra browser setup is
needed on `ubuntu-latest`.

## Local run

```sh
npm install -g @axe-core/cli
(cd site && python3 -m http.server 8000) &
axe http://localhost:8000/ http://localhost:8000/about.html --exit
```

## Adding pages

Add the new URL to the `axe` invocation in `.github/workflows/a11y.yml`, or
pass a sitemap with `--sitemap http://localhost:8000/sitemap.xml`.

## Caveats: where axe-cli gets harder

This POC is deliberately minimal — a static site served by `http.server`,
scanned anonymously, one page at a time. Real apps hit `axe-cli`'s limits
quickly. The scenarios below typically mean dropping `axe-cli` in favor of
driving axe-core directly from a browser-automation framework (Playwright +
[`@axe-core/playwright`](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/playwright),
Cypress + [`cypress-axe`](https://github.com/component-driven/cypress-axe), or
Selenium + [`@axe-core/webdriverjs`](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/webdriverjs)).

### Authentication / login-gated pages

`axe-cli` opens each URL in a fresh headless Chrome session with no way to
submit a form, set cookies, or attach an `Authorization` header. Anything
behind a login is invisible to it.

Options:

- **Test-only bypass.** Expose a non-prod-only "login as test user" endpoint
  that sets the session cookie, then hit pages directly. Cheapest, but
  requires app changes and a clear guard against shipping the bypass to
  production.
- **Switch to Playwright/Cypress.** Script the real login flow once, reuse
  the storage state, then call `AxeBuilder(page).analyze()` on each page.
  This is the standard answer for any non-trivial app.
- **Pre-baked session.** If the app accepts a long-lived token, inject it
  with a proxy like `mitmproxy` in front of `http.server`/the app and let
  `axe-cli` scan as normal. Brittle, but occasionally useful for
  header-auth APIs that render HTML.

### Single-page apps (SPAs) and client-rendered content

`axe-cli` scans after `load`, but SPAs often finish hydrating, fetching data,
or rendering route content well after that event. You can get false greens
(content not yet in the DOM) or false reds (loading spinners flagged).

Options:

- Use `--show-errors` and `--timeout` to give the page longer, but this is a
  blunt tool.
- Drive with Playwright/Cypress instead and `await` a known "page ready"
  signal (network idle, a specific selector, a test-only `window.__ready`
  flag) before running axe.

### Routes that need specific state (forms, modals, wizards)

A11y bugs often hide in interactive states — an open modal, an invalid form
field showing an error message, a disclosure widget expanded. `axe-cli` only
sees the initial render.

Options:

- Add dedicated test routes/URLs that render the component in the desired
  state (Storybook stories work well; `@storybook/test-runner` has built-in
  axe integration).
- Switch to Playwright/Cypress and interact with the page before calling
  axe.

### Per-page rules, ignores, and known issues

`axe-cli` accepts `--tags`, `--rules`, and `--disable`, but applies them
globally to every URL in the run. There is no per-URL config and no way to
mark a specific violation as "known, tracked elsewhere."

Options:

- Split into multiple workflow steps, each with its own `axe` invocation
  and rule set.
- Move to a framework integration where you can configure axe per test and
  use `.exclude()` selectors or a baseline file.

### Baselining / preventing regressions without fixing everything first

`--exit` is all-or-nothing: any violation fails the build. There is no
built-in baseline that says "fail only on *new* violations."

Options:

- Diff the JSON report against a committed baseline in a follow-up step
  (a few dozen lines of `jq`/Node). Workable but homegrown.
- Use [`axe-core-maven-html`](https://github.com/dequelabs/axe-core-maven-html)
  or third-party tools (Pa11y CI has thresholds; Deque's hosted
  axe Monitor/DevTools handles baselines first-class).

### Crawling a real site

`--sitemap` works if the app ships a sitemap, but `axe-cli` does not crawl
links. Large sites need either a maintained URL list or an external crawler.

Options:

- Generate a sitemap at build time and feed it to `--sitemap`.
- Use [Pa11y CI](https://github.com/pa11y/pa11y-ci) which supports a config
  file with a URL list, or a crawler like `wget --spider` to produce one.

### iframes, shadow DOM, cross-origin content

axe-core itself handles same-origin iframes and shadow DOM, but cross-origin
iframes (embedded YouTube, Stripe, etc.) are not introspectable from the
parent — axe will report what it can see and skip the rest. There is no
`axe-cli` flag that changes this; it is a browser security boundary.

Options:

- Test the embedded content separately at its own origin where possible.
- Accept that third-party widgets are out of scope and document it.

### Chrome / ChromeDriver version drift

We already hit this once in CI. `@axe-core/cli` bundles a pinned chromedriver
that drifts from the Chrome shipped on `ubuntu-latest` runners. The workflow
runs `npx browser-driver-manager install chrome` before each scan to
re-sync them; if that ever fails, pass `--chromedriver-path` explicitly to
`axe`.

### Performance at scale

Each URL spins up a fresh headless Chrome. A few dozen pages is fine; a few
hundred starts to dominate CI time and there is no parallelism flag.

Options:

- Shard the URL list across matrix jobs in the workflow.
- Switch to Playwright with multiple workers, which reuses browser contexts
  and parallelizes naturally.
