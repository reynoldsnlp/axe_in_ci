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
