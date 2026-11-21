# CBOM inventory action

Produce a [CycloneDX](https://cyclonedx.org/capabilities/cbom/) CBOM for a repository, and a
Markdown self-check report derived from it.

**This is an inventory, not an assessment.** It lists which cryptographic algorithms
appear in the source, and where. It does not score, gate, certify, or decide whether
anything "passes".

## Usage

```yaml
name: CBOM
on:
  workflow_dispatch:
  push:
    branches: [main]
    paths: ["src/**", "packages/**"]

permissions:
  contents: read

jobs:
  inventory:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: Lennonhaha/cbom-scan-action@v0
        with:
          version: "0.3.1"
```

Two artifacts come out: `app.cbom.json` (CycloneDX 1.7) and `self-check.md`
(also rendered into the job summary, first 40 lines).

## Inputs

| input | default | meaning |
|---|---|---|
| `path` | `.` | directory to scan, relative to the workspace |
| `out-dir` | `cbom-out` | where the CBOM, report and scan log go |
| `redact` | `true` | replace absolute paths with `<redacted>` in the report |
| `version` | `0.3.1` | `cbom-tools` version to run — pin it |
| `node-version` | `22` | Node used to run the tool |
| `upload-artifacts` | `true` | upload CBOM + report + log |

## Outputs

| output | meaning |
|---|---|
| `cbom-path` | path to the generated CBOM |
| `report-path` | path to the generated Markdown report |

## Why there is no `fail-on`

Every scanner action in this space offers `fail-on: high`. This one does not, on
purpose.

A threshold on a name-matching report turns a checklist into a verdict. The report
contains false positives by construction — prose in comments, compound names such as
`ML-DSA`, KAT vectors that look like keys — and a job that goes red on them teaches
the team to add `continue-on-error`, which then also hides the real findings. The
report is meant to be read by a person; the decision to block belongs to a reviewer,
not to a regex.

## Why `redact` defaults to `true`

CI logs and artifacts outlive the run and are often visible beyond the team. An
absolute path carries a username, a drive letter, an internal directory layout. None
of that is needed to act on a finding: the relative path and the file name are.
The cost is negligible (1.4 s vs 1.5 s on a 1.9 M-line tree).

## What the report can and cannot say

Can: the algorithms present, their call sites (file, line, symbol), whether any of
them are deprecated (MD5, SHA-1, DES, DSA), whether PEM private-key blocks appear in
source, and an explicit "cannot determine" for data-at-rest encryption.

Cannot: whether a protocol is actually enabled, whether a key is 1024 or 2048 bits,
whether a module is certified, whether a policy is followed. Those are the questions
a 密评 assessor asks on site; a static inventory is the input to that conversation,
not the answer.

## Pinning

`version` is an input rather than a floating `latest`. Two reports are only comparable
when the same tool version produced them; an action that silently upgrades changes
the meaning of a stored artifact.

## License

Apache-2.0. The tool it runs, [`cbom-tools`](https://www.npmjs.com/package/cbom-tools),
is Apache-2.0 as well, with zero runtime dependencies and no network access at scan
time.
