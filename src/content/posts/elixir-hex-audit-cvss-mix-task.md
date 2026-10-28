---
title: "Building mix hex.audit_cvss: CVSS-Scored Dependency Auditing for Elixir"
author: "Pikender Sharma"
pubDatetime: 2026-10-28T10:00:00Z
description: "How I built a Mix task that pulls pre-computed CVSS 4.0 scores from Hex.pm's API to give Elixir projects a CI-friendly, multi-format dependency security audit — and what I learned about the gap in existing tools along the way."
tags: ["elixir", "security", "dependencies", "mix", "cvss", "hex", "devops"]
featured: false
draft: false
---

Dependency security scanning is table stakes for any production application. For Elixir projects, the obvious first stop is `mix hex.audit` — it's built in, it works, and it covers hundreds of advisories. But it outputs to the terminal and exits 0 whether or not vulnerabilities exist. That makes it less effective in CI.

This post walks through building `mix hex.audit_cvss`, a Mix task that fills that gap: CVSS 4.0 scores, prioritized output, a CI gate (exit 1 on findings), and four export formats including SARIF for GitHub's Security tab. ([Full source on GitHub Gist](https://gist.github.com/pikender/c21dcdbd24ab720f25832b3f5a6ad415))

## The Problem with Existing Tools

Before writing anything, I surveyed what already existed.

**`mix hex.audit`** (built into Hex.pm) is the right baseline. It reads a live 350+ advisory feed from OSV.dev, covers retired packages, and requires no extra install. But it only prints to the terminal and always exits 0 — you can't block a CI pipeline on it.

**`mix_audit`** mirrors a static GitHub advisory snapshot and has an ~90% miss rate versus `mix hex.audit`. It doesn't cover the full range of Hex advisories, making it insufficient for comprehensive vulnerability scanning.

**Trivy** supports `mix.lock` but only reads the GitHub Advisory Database (GHSA). The Erlang Ecosystem Foundation publishes advisories under an EEF-CVE namespace that has no GHSA equivalent — Trivy misses roughly half of Hex advisories.

**Snyk** doesn't parse `mix.lock` at all. That was the original reason the team wrote a standalone `security.exs` script as a first pass.

**`sobelow`** is a different tool entirely — it does static analysis of your own Phoenix code for XSS, SQL injection, CSRF, and similar. Complementary, not competing.

The gap: a tool that reads `mix.lock`, hits a live advisory feed, surfaces CVSS scores for prioritization, and exits 1 in CI when findings exist.

## The Key Discovery: Hex.pm Already Has CVSS Scores

While exploring whether to wrap `mix hex.audit`, noticed that the Hex.pm release API response includes a `security_advisories` array — richer than what the CLI surfaces:

```
GET https://hex.pm/api/packages/{name}/releases/{version}
```

Each advisory object carries:

```json
{
  "id": "HEXPM-1234",
  "cvss_score": 8.7,
  "cvss_rating": "high",
  "cvss_vector": "CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:N/VI:N/VA:H/SC:N/SI:N/SA:N",
  "summary": "Unbounded HPACK integer decoding allows DoS",
  "affected_versions": "< 1.0.4"
}
```

`mix hex.audit` queries these same endpoints but ignores `cvss_score`, `cvss_rating`, and `cvss_vector`. It only surfaces the advisory ID and summary.

This was the pivotal insight: CVSS scores didn't need to be computed locally — Hex.pm had already done it. The team's standalone `security.exs` script included a 670-line CVSS 4.0 scorer ported from a JavaScript reference implementation. That entire module was unnecessary for Hex packages.

## What `mix.lock` Gives You

The lockfile is the right source of truth. It lists every package that will actually run in production — direct and transitive. My project had 27 entries: 14 declared in `mix.exs` and 13 transitive dependencies.

`hpax`, which had an 8.7 HIGH advisory, isn't in `mix.exs` at all. It's pulled in by `mint`, which is pulled in by `finch`. Scanning only declared dependencies would have missed it.

Parsing `mix.lock` in Elixir is straightforward — the file is a valid Elixir term:

```elixir
defp read_lock_file do
  File.read!("mix.lock")
  |> Code.eval_string()
  |> elem(0)
  |> Enum.filter(fn {_name, spec} ->
    # Only Hex packages — skip git/path deps
    is_tuple(spec) and elem(spec, 0) == :hex
  end)
  |> Enum.map(fn {name, spec} ->
    {name, elem(spec, 2)}  # {package_name, version}
  end)
end
```

## Real Findings in This Codebase

Running against **my project** returned 18 advisories across 6 packages:

| Score | Rating | Package             | Summary                                    |
| ----- | ------ | ------------------- | ------------------------------------------ |
| 8.7   | HIGH   | `hpax 1.0.3`        | Unbounded HPACK integer decoding DoS       |
| 8.2   | HIGH   | `plug 1.19.2`       | Multipart header buffer DoS                |
| 6.9   | MED    | `decimal 3.1.1`     | `Decimal.round/3` unbounded allocation DoS |
| 6.9   | MED    | `mint 1.9.0`        | HTTP/1 chunked framing smuggling           |
| 5.9   | MED    | `postgrex 0.22.2`   | SQL injection via `:comment` in stream     |
| 5.1   | MED    | `plug_crypto 2.1.1` | Secret key base in stack trace             |

Without scores, this is a flat list with no ordering. With CVSS 4.0, it's a prioritized queue: fix `hpax` and `plug` this sprint, schedule the medium findings.

## The Mix Task

The task lives at `lib/mix/tasks/hex/audit_cvss.ex` with module name `Mix.Tasks.Hex.AuditCvss`, invoked as `mix hex.audit_cvss`.

The module naming convention maps directly to the invocation: dots in the task name become submodule separators, and CamelCase is applied to each segment. `Mix.Tasks.Hex.AuditCvss` → `mix hex.audit_cvss`.

The core behavior:

1. Read `mix.lock`, extract Hex packages
2. For each package, `GET hex.pm/api/packages/{name}/releases/{version}`
3. Collect advisories and retirement status
4. Write the requested output format(s)
5. Call `Mix.raise/1` if any findings exist — this exits with code 1

`Mix.raise/1` is the CI gate. It's the difference between a tool that reports and a tool that blocks.

### Handling Dependencies

The task needs two dev-only packages:

```elixir
{:req,   "~> 0.5", only: :dev, runtime: false}
{:jason, "~> 1.0", only: :dev, runtime: false}
```

`req` handles HTTP. `jason` is required for the JSON, SARIF, and CycloneDX formats — CSV works without it. Both are checked at runtime with `Code.ensure_loaded?/1` before use, so a missing dep gives a clear error with the exact snippet to add rather than a cryptic function-not-found crash.

### `--verbose` flag

By default the task prints a compact summary grouped by severity. Pass `--verbose` for per-package progress and a full advisory list sorted by CVSS score descending:

```bash
$ mix hex.audit_cvss --verbose
[1/27] bandit 1.6.7
[2/27] decimal 3.1.1
  ! 1 advisory(ies)
...

Advisories (sorted by CVSS score descending):
  [8.7 HIGH] hpax 1.0.3 — HEXPM-123
    Unbounded HPACK integer decoding allows DoS
  [8.2 HIGH] plug 1.19.2 — HEXPM-456
    Multipart header buffer DoS
  ...
```

### Output files are gitignored

All four output filenames match the `hex_audit*` pattern added to `.gitignore`. Audit results are point-in-time snapshots — they belong in CI artifacts, not in the repo.

## Four Output Formats

The initial version wrote CSV. The final version supports four formats, selected with `--format`:

```bash
mix hex.audit_cvss                      # hex_audit.csv
mix hex.audit_cvss --format json        # hex_audit.json
mix hex.audit_cvss --format sarif       # hex_audit.sarif
mix hex.audit_cvss --format cyclonedx  # hex_audit.cdx.json
```

**CSV** is the default — easiest to open in a spreadsheet for team review.

**JSON** uses proper types: arrays for `aliases` and `affected_versions`, float for `cvss_score`, boolean for `retired`. Useful for custom downstream pipelines.

**SARIF** (Static Analysis Results Interchange Format) is what GitHub renders natively in the Security tab. Upload the file as an artifact in CI and GitHub shows inline annotations sorted by severity — no third-party integration needed. CVSS ratings map to SARIF levels: `critical/high → error`, `medium → warning`, `low → note`.

```yaml
# .github/workflows/audit.yml
- run: mix hex.audit_cvss --format sarif || true
- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: hex_audit.sarif
```

**CycloneDX** (OWASP's SBOM standard) is ingestible by Dependency-Track and similar tools. The format separates `components` (all 27 dependencies, as a bill of materials) from `vulnerabilities` (only affected ones), linked via package URL:

```
pkg:hex/mint@1.9.0
```

CVSS scores nest at `vulnerabilities[].ratings[].score` with `"method": "CVSSv4"` — the spec allows multiple rating sources per vulnerability.

Multiple formats can be requested in a single run — `--format` can be repeated and Hex.pm is queried only once:

```bash
mix hex.audit_cvss --format sarif --format cyclonedx --format csv
```

## Compared to the Standalone Script

The team had a working `scripts/security.exs` before this task existed. The comparison:

|                           | `scripts/security.exs` | `mix hex.audit_cvss`        |
| ------------------------- | ---------------------- | --------------------------- |
| Data source               | OSV.dev directly       | Hex.pm release API          |
| CVSS scoring              | 670-line local scorer  | Pre-computed by Hex.pm      |
| Retired packages          | No                     | Yes                         |
| Output formats            | CSV only               | CSV, JSON, SARIF, CycloneDX |
| CI gate (exit 1)          | No                     | Yes                         |
| Works outside Mix project | Yes                    | No                          |

The scripts serve different purposes. The standalone script is portable — you can run it in any directory with `elixir scripts/security.exs` and no Mix project context. The Mix task is the right tool inside a project's CI pipeline.

A companion `scripts/licenses.exs` rounds out the picture: it queries the Hex.pm package API (not the release endpoint) for license metadata on every dep in `mix.lock`, writing a `licenses.csv` with package, version, license, and repo link. Together the three tools cover security advisories, CVSS prioritization, and license compliance — all from `mix.lock` as the single source of truth.

## The Result

```bash
$ mix hex.audit_cvss
Auditing 27 Hex packages...

CRITICAL: 0 packages
HIGH:     2 packages (hpax, plug)
MEDIUM:   4 packages (decimal, mint, postgrex, plug_crypto)
LOW:      0 packages

Written: hex_audit.csv
** (Mix) mix hex.audit_cvss found 18 security advisories. Review hex_audit.csv.
```

Exit code 1. Pipeline blocked. Job done.

---

[Full source on GitHub Gist](https://gist.github.com/pikender/c21dcdbd24ab720f25832b3f5a6ad415)
