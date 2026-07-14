<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2026 The Linux Foundation
-->

# 🧪 Go Test Action

<!-- prettier-ignore-start -->
<!-- markdownlint-disable-next-line MD013 -->
[![Linux Foundation](https://img.shields.io/badge/Linux-Foundation-blue)](https://linuxfoundation.org/) [![Source Code](https://img.shields.io/badge/GitHub-100000?logo=github&logoColor=white&color=blue)](https://github.com/lfreleng-actions/go-test-action) [![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![pre-commit.ci status badge]][pre-commit.ci results page] [![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/lfreleng-actions/go-test-action/badge)](https://scorecard.dev/viewer/?uri=github.com/lfreleng-actions/go-test-action)
<!-- prettier-ignore-end -->

Runs `go test` with coverage and race detection support.

## go-test-action

This action wraps `actions/setup-go` and `go test` behind a validated
interface. Coverage collection and the race detector default on. The
`permit_fail` input supports a non-blocking mode that reports test
failures through outputs and the step summary without failing the
job, matching the soft-fail semantics used across this organisation's
actions.

## Usage Example

<!-- markdownlint-disable MD046 -->

```yaml
steps:
  - name: "Run tests"
    id: test
    uses: lfreleng-actions/go-test-action@main
    with:
      path_prefix: '.'
```

Informational (non-blocking) test run with a coverage artifact:

```yaml
steps:
  - name: "Run tests"
    id: test
    uses: lfreleng-actions/go-test-action@main
    with:
      permit_fail: 'true'
      artifact_upload: 'true'
      artifact_name: coverage-report
```

<!-- markdownlint-enable MD046 -->

## Requirements

The action needs `realpath` (GNU coreutils) and `awk` on the runner.
GitHub-hosted Ubuntu runners include them; minimal self-hosted or
non-Linux runners must provide them. The action installs the Go
toolchain via the pinned `actions/setup-go`, with caching off per
organisation security policy, so runners need egress to the Go
distribution endpoints and, for module downloads during the test
build, to `proxy.golang.org` and `sum.golang.org`.

## Inputs

<!-- markdownlint-disable MD013 -->

| Name            | Required | Default       | Description                                                                                           |
| --------------- | -------- | ------------- | ----------------------------------------------------------------------------------------------------- |
| go_version      | False    | `''`          | Explicit Go version to install; takes precedence over go_version_file                                 |
| go_version_file | False    | `go.mod`      | File to read the Go version from, relative to path_prefix                                             |
| path_prefix     | False    | `.`           | Project directory; must resolve within the workspace                                                  |
| coverage        | False    | `true`        | Collect coverage with `-coverprofile`                                                                 |
| race            | False    | `true`        | Enable the race detector                                                                              |
| test_args       | False    | `''`          | Extra `go test` flags (restricted character set)                                                      |
| test_target     | False    | `./...`       | Space-separated package pattern(s) to test                                                            |
| timeout         | False    | `10m`         | Go test timeout as a Go duration string, e.g. `10m` or `1h30m`                                        |
| make_args       | False    | `''`          | When set, run the project's Makefile via make-action instead of the built-in `go test` (see below)    |
| permit_fail     | False    | `false`       | Report test failures without failing the action                                                       |
| artifact_upload | False    | `false`       | Upload the coverage profile as an artifact; requires coverage; skips when the run produced no profile |
| artifact_name   | False    | `go-coverage` | Artifact name (restricted character set)                                                              |

<!-- markdownlint-enable MD013 -->

## Outputs

<!-- markdownlint-disable MD013 -->

| Name             | Description                                                                                                      |
| ---------------- | ---------------------------------------------------------------------------------------------------------------- |
| tests_passed     | Whether the test run passed: `true` or `false`                                                                   |
| coverage_percent | Total coverage percentage from `go tool cover`; empty when coverage is off, `0` when the run produced no profile |

<!-- markdownlint-enable MD013 -->

The action emits `tests_passed` for every run, including failing runs
in both `permit_fail` modes, so downstream steps can branch on the
result. `coverage_percent` carries the `total:` figure from
`go tool cover -func`. It stays empty when coverage is off and
reports `0` when the test build produced no profile (for example a
compilation failure under `permit_fail: true`), so numeric consumers
keep working. When a profile exists but `go tool cover` fails or
emits no `total:` line, the action fails with a clear error rather
than continuing with an empty value.

## Input Constraints

The action validates every input before use and fails closed on
anything outside the expected form:

- `test_args` accepts the characters
  `A-Z a-z 0-9 space . _ / = , : @ + -` and nothing else, which
  excludes shell metacharacters such as backticks, `$(`, `;`, `&`,
  `|` and newlines
- `test_target` accepts `A-Z a-z 0-9 space . _ / -` and rejects
  tokens starting with `-`, so package patterns cannot smuggle extra
  flags into the test command
- `timeout` must parse as a Go duration built from digit+unit groups
  (`ns`, `us`, `ms`, `s`, `m`, `h`), for example `10m` or `1h30m`
- `artifact_name` accepts `A-Z a-z 0-9 . _ -`; booleans accept
  `true` or `false`

## Path Constraints

Relative values for `path_prefix` resolve against
`GITHUB_WORKSPACE`, not the current working directory, so behaviour
stays deterministic when a calling workflow sets a custom working
directory. The project directory must resolve within
`GITHUB_WORKSPACE`, and the version file must resolve within the
project directory (`path_prefix`); paths that escape these
boundaries fail the action. The coverage profile writes to
`<path_prefix>/coverage.out`.

## Soft-Fail Mode

With `permit_fail: true`, a failing test run reports through the
step summary, emits `tests_passed: false` and exits with a zero
status, so the calling job continues. Use this for informational test lanes
(for example, race detection on legacy codebases) where the
organisation gates merges on a separate blocking lane. With the
default `permit_fail: false`, a failing run still emits its outputs
and summary before failing the action.

## Monorepo and Nested Module Support

Point `path_prefix` at the directory containing the module's
`go.mod`. For legacy modules with ancient `go` directives (for
example Jenkins-era Gerrit projects on `go 1.16`), pass an explicit
modern `go_version`:

<!-- markdownlint-disable MD046 -->

```yaml
steps:
  - name: "Test nested module"
    uses: lfreleng-actions/go-test-action@main
    with:
      path_prefix: 'src/k8splugin'
      go_version: '1.25.11'
      test_target: './internal/...'
```

<!-- markdownlint-enable MD046 -->

## Makefile-Driven Tests (`make_args`)

Some projects must run build steps before `go test`, for example
compiling a Go plugin that the test suite loads at runtime (ONAP
`multicloud/k8s` builds a mock plugin in its `test` target). For
these, set `make_args` to delegate the whole run to the project's own
Makefile through the pinned
[`make-action`](https://github.com/lfreleng-actions/make-action). The
action runs `make -C <path_prefix> <make_args>` and skips the
built-in `go test` invocation:

<!-- markdownlint-disable MD046 -->

```yaml
steps:
  - name: "Build plugin and run tests via make"
    uses: lfreleng-actions/go-test-action@main
    with:
      path_prefix: 'src/k8splugin'
      go_version: '1.25.11'
      make_args: 'test'
```

<!-- markdownlint-enable MD046 -->

On this path the `coverage`, `race`, `test_args`, `test_target` and
`timeout` inputs do not apply; the Makefile controls how tests run,
and `coverage_percent` stays empty. `permit_fail` still applies: a
failing `make` run reports through the step summary and emits
`tests_passed: false` without failing the action when
`permit_fail: true`. `make_args` accepts the same character allowlist
as `test_args` and must not contain `-C`/`--directory`; the action
derives the working directory from `path_prefix` and passes it as
`make -C <path_prefix>` itself.

## Implementation Details

<!-- markdownlint-disable MD013 -->

1. **Input Validation**: Validates booleans, flag character allowlists, the timeout duration format, artifact naming and path boundaries before use
2. **Toolchain Setup**: Installs Go via the pinned `actions/setup-go` with `cache: false` (organisation security stance on cache poisoning)
3. **Test Run**: Runs `go test` in the project directory with the requested race/coverage flags (`-covermode atomic`) and timeout, then parses the total coverage from `go tool cover -func`. Setting `make_args` skips this step and instead runs the pinned `make-action` as `make -C <path_prefix> <make_args>`
4. **Outputs and Summary**: Emits pass/fail and coverage outputs plus a step summary; the optional artifact upload uses the pinned `actions/upload-artifact` and skips when the run produced no coverage profile

<!-- markdownlint-enable MD013 -->

## Notes

- Coverage collection uses `-covermode atomic`, which stays correct
  when combined with the race detector
- The race detector needs cgo and a supported host platform; the
  action leaves `CGO_ENABLED` untouched for test runs

[pre-commit.ci results page]: https://results.pre-commit.ci/latest/github/lfreleng-actions/go-test-action/main
[pre-commit.ci status badge]: https://results.pre-commit.ci/badge/github/lfreleng-actions/go-test-action/main.svg
