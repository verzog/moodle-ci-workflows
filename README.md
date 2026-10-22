# Moodle CI workflows

Shared GitHub Actions workflows for Vernon Spain's Moodle plugins. Each plugin repository calls these instead of
keeping its own copy, so a CI fix or a new Moodle branch is made once, here.

| Workflow | What it does |
|---|---|
| `moodle-plugin-ci.yml` | Runs moodle-plugin-ci 4 (lint, code checker, PHPDoc, validate, savepoints, Mustache, Grunt, PHPUnit, Behat) across Moodle 5.1–5.3 LTS and Moodle `main` |
| `release-zip.yml` | When a GitHub release is published, checks the tag matches `$plugin->release`, builds `<component>-<tag>.zip` and attaches it |

## Using them in a plugin

`.github/workflows/moodle-ci.yml`:

```yaml
name: Moodle Plugin CI

# Run on pull requests, and on pushes to main and tags only.
on:
  push:
    branches:
      - main
    tags:
      - '*'
  pull_request:

permissions:
  contents: read

jobs:
  ci:
    uses: verzog/moodle-ci-workflows/.github/workflows/moodle-plugin-ci.yml@v1
```

`.github/workflows/release.yml`:

```yaml
name: Release zip

on:
  release:
    types: [published]

permissions:
  contents: write

jobs:
  release:
    uses: verzog/moodle-ci-workflows/.github/workflows/release-zip.yml@v1
```

Keep the plugin's `.gitattributes` marking `/.github`, `/tests`, `/.gitattributes` and `/.gitignore` as
`export-ignore`, so they stay out of the release zip.

## CI inputs

All are optional. Pass them under `with:`.

| Input | Default | Use |
|---|---|---|
| `matrix` | Moodle 5.1–5.3 LTS and Moodle `main` (see below) | A JSON list of `{moodle-branch, php, database}` cells, for a plugin that needs a different range |
| `phplint-php` | `'8.2'` | The PHP version that runs the syntax check: the lowest PHP in the matrix |
| `grunt-branch` | `MOODLE_501_STABLE` | The branch that runs the Grunt check, which must be the branch AMD builds are made on |
| `behat` | `true` | Set to `false` for a plugin with no Behat features |
| `behat-tags` | empty | Only run scenarios with these tags, e.g. `'@local_myplugin'` |
| `timezone` | `Australia/Sydney` | Time zone for the test runs |

The default matrix:

| Moodle | PHP | Database |
|---|---|---|
| MOODLE_501_STABLE | 8.2 | PostgreSQL 17 |
| MOODLE_501_STABLE | 8.4 | MariaDB 11 |
| MOODLE_502_STABLE | 8.3 | PostgreSQL 17 |
| MOODLE_503_STABLE | 8.3 | PostgreSQL 17 |
| MOODLE_503_STABLE | 8.4 | MariaDB 11 |
| main | 8.4 | PostgreSQL 17 (allowed to fail) |

MOODLE_500_STABLE and older are deliberately left out: the plugins require Moodle 5.1 or later. Moodle 5.2 and
later have no PHP 8.2 cell because they dropped PHP 8.2.

Example, for a plugin without Behat tests:

```yaml
jobs:
  ci:
    uses: verzog/moodle-ci-workflows/.github/workflows/moodle-plugin-ci.yml@v1
    with:
      behat: false
```

Warnings fail the build: the code checker, PHPDoc and Grunt run with zero warnings allowed, and PHPUnit with
`--fail-on-warning`. Proprietary (Mode A) plugins keep their `.phpcs.xml` boilerplate exclusion in their own
repository; the workflow picks it up from there.

## Release input

| Input | Default | Use |
|---|---|---|
| `plugin-dir` | the part of `$plugin->component` after the first underscore | The zip's top folder, which must be the plugin's directory name (`local_myplugin` → `myplugin/`) |

## Versions

Plugins call `@v1`, which is a **branch**. It only ever moves forward to the newest compatible change on `main`, so
fixes reach every plugin on its next run. A change that needs plugins to change their caller (for example, a
removed input) goes on a new branch, `v2`, and plugins move to it one at a time.

To change a workflow:

1. Open a pull request here. The "Lint workflows" check runs actionlint and shellcheck over the files.
2. Test the change from one plugin by pointing its caller at the pull request's branch (`@my-branch`), and check
   that plugin's CI.
3. Merge, then move `v1` up to `main`. This is a fast-forward, never a force-push:

   ```sh
   git fetch origin
   git push origin origin/main:refs/heads/v1
   ```

To pin a plugin to one exact version while testing, use the commit SHA instead of `@v1`.

## Licence

GNU GPL v3 or later. See [LICENSE](LICENSE).
