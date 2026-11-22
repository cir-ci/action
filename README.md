# CirCI GitHub Action

Uploads screenshots (and built static sites) from a GitHub Actions workflow to your [CirCI](https://circi.dev) server, a self-hosted visual regression testing service. It runs [`@circi/cli`](https://www.npmjs.com/package/@circi/cli) for you, sets outputs for the build, and writes a job summary with a link to it.

Workflow examples, the first build and the merge gate are at [circi.dev/ci/ci-examples/](https://circi.dev/ci/ci-examples/); the CirCI check on pull requests is at [circi.dev/integrations/github/](https://circi.dev/integrations/github/).

## Usage

Store your project's CI token as a repository secret named `CIRCI_TOKEN` and your server's address as a repository variable named `CIRCI_API_URL`. Then, after the step that writes your screenshots:

```yaml
name: Visual tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  visual:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test # writes ./screenshots

      - uses: cir-ci/action@v1
        with:
          path: ./screenshots
          token: ${{ secrets.CIRCI_TOKEN }}
          api-url: ${{ vars.CIRCI_API_URL }}
```

Run it on `push` to your default branch as well as on pull requests: pull requests are compared with the latest approved build of the default branch.

### Gate the merge with `wait`

With `wait: true` the step waits for the build to be processed and fails the job while visual changes need review and this job is the gate:

```yaml
- uses: cir-ci/action@v1
  with:
    path: ./screenshots
    token: ${{ secrets.CIRCI_TOKEN }}
    api-url: ${{ vars.CIRCI_API_URL }}
    wait: true
```

When the project is linked to the repository, CirCI's own check is the gate and the job passes (see `gate`). Approve the changes in the dashboard, then re-run the job.

### Shard a slow suite

Give each job its number and the total. The shards of one workflow run join into one build, which is processed once the last one arrives:

```yaml
jobs:
  visual:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      # ... install, then run this shard's tests
      - run: npx playwright test --shard=${{ matrix.shard }}/4
      - uses: cir-ci/action@v1
        with:
          path: ./screenshots
          token: ${{ secrets.CIRCI_TOKEN }}
          api-url: ${{ vars.CIRCI_API_URL }}
          shard: ${{ matrix.shard }}
          shards: 4
          wait: true
```

Keep `fail-fast: false`, so one failed job doesn't cancel the shards the build is waiting for. Re-run all the jobs of a workflow, not only the failed one.

### Host a Storybook

`site` uploads a built static site, which CirCI hosts next to the build:

```yaml
- run: npm run build-storybook
- uses: cir-ci/action@v1
  id: storybook
  with:
    site: storybook-static
    site-name: storybook
    token: ${{ secrets.CIRCI_TOKEN }}
    api-url: ${{ vars.CIRCI_API_URL }}
- run: echo "Storybook is at ${{ steps.storybook.outputs.site-url }}"
```

Give `path` as well to upload screenshots and the site in one step. With shards, upload the site from its own job, not from every shard.

## Inputs

| Input               | Default          | What it does                                                                                 |
| ------------------- | ---------------- | -------------------------------------------------------------------------------------------- |
| `path`              |                  | Directory of `.png` screenshots, searched recursively. Required unless `site` is set.        |
| `token`             | `$CIRCI_TOKEN`   | The project's CI token. Pass `${{ secrets.CIRCI_TOKEN }}`.                                   |
| `api-url`           | `$CIRCI_API_URL` | Your CirCI address, for example `https://ci.acme.io`.                                        |
| `name`              | `default`        | Build name, for several kinds of builds per commit.                                          |
| `wait`              | `false`          | Wait for the result and fail the job while changes need review and this job is the gate.     |
| `gate`              | server decides   | `status` (the git host's check is the gate, the job passes) or `exit-code` (this job is).    |
| `timeout`           | `600`            | How many seconds `wait` waits.                                                               |
| `shard`, `shards`   |                  | This job's number (from 1) and how many jobs there are. `shard: 2/4` also works.             |
| `site`              |                  | A built static site to host, for example `storybook-static`. Uploaded after the screenshots. |
| `site-name`         | `site`           | Which of the project's sites this is.                                                        |
| `site-kind`         | `auto`           | `storybook`, `vitest-report`, `playwright-report`, `coverage`, `docs`, `generic` or `auto`.  |
| `cli-version`       | `0.2.0`          | The `@circi/cli` version to run.                                                             |
| `working-directory` | `.`              | Where to run. `path` and `site` are relative to it.                                          |

The commit, branch, pull request number and base branch come from the GitHub Actions environment; the action passes none of them.

`cli-version` is pinned, so a CLI release never changes your builds; a new release of this action moves it. Set it yourself (`latest` works) to run another version.

## Outputs

| Output                        | What it is                                                                   |
| ----------------------------- | ---------------------------------------------------------------------------- |
| `build-url`                   | The build's address in the dashboard.                                        |
| `build-id`                    | The build's ID.                                                              |
| `build-number`                | The build's number in its project.                                           |
| `status`                      | `uploaded` (not waited for), `passed`, `needs-review`, `error` or `skipped`. |
| `changed`, `added`, `removed` | Screenshot counts, with `wait`.                                              |
| `site-url`                    | The commit address of the uploaded site.                                     |

The job summary shows the build link, the status and the counts.

```yaml
- uses: cir-ci/action@v1
  id: circi
  with:
    path: ./screenshots
    token: ${{ secrets.CIRCI_TOKEN }}
    api-url: ${{ vars.CIRCI_API_URL }}
- run: echo "Build ${{ steps.circi.outputs.build-url }} is ${{ steps.circi.outputs.status }}"
```

## Pull requests from forks

GitHub gives no secrets to workflows triggered by a pull request from a fork (or by Dependabot), so `token` is empty there. The action then skips the upload with a notice instead of failing, sets `status` to `skipped` and says so in the job summary. Without a token on any other run it skips with a warning, which is the sign that the secret isn't set. A missing `path` or `api-url` still fails the step.

Don't switch to `pull_request_target` to get the secret: it runs with your secrets on code from the fork.

## Requirements

- A CirCI server that GitHub's runners can reach.
- Node 22 or newer. GitHub's hosted runners have it. If the runner has none, or an older one, the action sets up Node 24 itself.
- `npx` to fetch the CLI from npm.

Windows, macOS and Linux runners work; the action's steps use `bash`.

## Exit codes

The step fails with the CLI's exit code: `1` when `wait` is on, changes need review and this job is the gate; `2` for a setup or upload error, a build that couldn't be processed, or `wait` timing out. [Errors and exit codes](https://circi.dev/reference/errors-and-exit-codes/) lists the messages.

## Development

`node --test test/run.test.mjs` runs the script against a fake CirCI server (`test/fake-circi.mjs`) and the real `@circi/cli`; CI also runs the action itself on GitHub. Only the CirCI server is faked.

## License

GPL-3.0, like CirCI. See [LICENSE](LICENSE).
