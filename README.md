# Kata: Fail Fast on Invalid Prometheus Rule Files

A coding exercise based on Prometheus issue
[#11486, "parse alerting before loading WAL"](https://github.com/prometheus/prometheus/issues/11486).

## Purpose of This Kata

This exercise is for developers who are new to the Prometheus codebase.
You practice agentic coding in a very large Go codebase: you find the
relevant code, change it, and prove that the change works.

The functional change is small. Most of the work is in understanding how
Prometheus starts up and how it loads rule files.

## Starting Point

Work on the Prometheus repository at commit `2834a665e` ("Add support for
promoting all OTel resource attributes (#16426)"). This commit is the last
one before the upstream fix was merged.

```bash
git clone https://github.com/prometheus/prometheus.git
cd prometheus
git switch -c kata/parse-rules-before-wal 2834a665e
```

The codebase at this commit needs Go 1.23 or later.

## Background

Prometheus keeps recently ingested samples in a write-ahead log (WAL). At
startup, Prometheus replays the WAL to rebuild its in-memory state. On a
large server, WAL replay can take many minutes and needs a lot of memory
and disk I/O.

The main configuration file (`prometheus.yml`) can reference rule files
through the `rule_files` setting. Rule files contain recording rules and
alerting rules. Each entry in `rule_files` is a file path or a glob
pattern.

Prometheus already validates the main configuration file, and the scrape
configuration files it references, before it starts the TSDB and replays
the WAL. Earlier issues introduced this behavior
([#7322](https://github.com/prometheus/prometheus/issues/7322),
[#7399](https://github.com/prometheus/prometheus/pull/7399)). Rule files
are different: Prometheus parses them only after WAL replay has finished.

### The Incident Behind the Issue

An operator deployed a syntactically broken rule file. Prometheus ran
under a systemd unit with `Restart=on-failure`. Each start went through
this sequence:

1. Prometheus started and replayed the full WAL.
2. Prometheus parsed the rule files, found the error and exited with a
   non-zero exit code.
3. systemd restarted Prometheus.

The loop ran for several days and exhausted the disk. Each restart paid the
full cost of WAL replay only to fail on an error that could have been
found in milliseconds.

## Problem Statement

When a rule file that exists contains invalid content, Prometheus detects
the error only after the expensive WAL replay. A broken rule file therefore
causes slow failures and, under automatic restart, a resource-intensive
crash loop.

## Goal

Prometheus validates all existing rule files referenced by the configuration
before it starts the TSDB and replays the WAL. If a rule file is invalid,
Prometheus exits immediately with a clear error message.

## Non-Goals

- Do not change how Prometheus treats rule file patterns that match no
  files. Prometheus deliberately tolerates missing rule files: an operator
  can add a rule file later and trigger a reload with `SIGHUP`. This
  workflow must keep working.
- Do not change the behavior of configuration reloads at runtime (`SIGHUP`
  or the `/-/reload` endpoint). A reload with an invalid rule file keeps
  the current behavior.
- Do not change rule evaluation, rule file syntax or the `promtool` command.
- Do not change the behavior of agent mode. Agent mode does not support rule files.

## User Stories

- As an operator, I want Prometheus to reject an invalid rule file within
  seconds of startup, so that a bad deployment does not cost a full WAL
  replay on every restart attempt.
- As an operator, I want the startup error to name the configuration file,
  the broken rule file and the parse error, so that I can fix the problem
  without further investigation.
- As an operator, I want Prometheus to keep starting normally when a rule
  file pattern matches no files, so that I can keep adding rule files later
  and reloading the configuration.

## Functional Requirements

### FR-1: Validate Rule Files Before WAL Replay

At startup in server mode, Prometheus must resolve every pattern in
`rule_files` to the files that currently exist and parse each of these
files. This validation must complete before Prometheus starts the TSDB,
that is, before it logs `Starting TSDB ...`.

### FR-2: Same Validation as the Regular Rule Loading

The early validation must reject every rule file that the regular rule
loading after WAL replay would reject. This includes YAML syntax errors,
unknown fields, invalid PromQL expressions and other rule validation
errors. The early validation must not reject a rule file that the regular
rule loading accepts.

### FR-3: Exit on Invalid Rule Files

If at least one rule file is invalid, Prometheus must log an error and exit
with exit code `2`. This matches the existing behavior for an invalid main
configuration file.

### FR-4: Informative Error Message

The error log entry must follow the style of the existing startup check for
the main configuration file. The entry must contain:

- the value of `--config.file`,
- the absolute path of the configuration file,
- the path of the invalid rule file,
- the underlying parse error.

### FR-5: Tolerate Missing Rule Files

A pattern in `rule_files` that matches no files must not cause an error.
Startup continues normally.

### FR-6: Unchanged Behavior for Valid Configurations

If all rule files are valid, startup behaves as before: Prometheus replays
the WAL, loads the rules and becomes ready.

### FR-7: Automated Tests

Automated Go tests must cover the new validation. The tests must include at
least one valid rule file and at least one invalid rule file. All existing
tests in the affected packages must still pass.

## Acceptance Criteria

Each criterion uses a minimal `prometheus.yml` with a `rule_files` entry
that points to the rule file under test, and a fresh `--storage.tsdb.path`
directory unless stated otherwise.

### AC-1: Invalid Rule File Fails Fast

Given a rule file that contains an unknown field:

```yaml
groups:
  - name: invalid
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job)(rate(http_requests_total[5m]))
        unexpected_field: this_field_should_not_be_here
```

When I start Prometheus with this configuration,
then Prometheus exits with exit code `2`,
and the log does not contain `Starting TSDB ...`,
and the error log entry contains the value of `--config.file`, the path of
the invalid rule file and a message that names the unknown field.

### AC-2: Invalid PromQL Expression Fails Fast

Given a rule file whose `expr` is not a valid PromQL expression, for
example `sum(rate(http_requests_total[5m])`,
when I start Prometheus with this configuration,
then Prometheus exits with exit code `2` before it logs `Starting TSDB ...`,
and the error log entry contains the path of the rule file and the
PromQL parse error.

### AC-3: Invalid YAML Fails Fast

Given a rule file that is not valid YAML,
when I start Prometheus with this configuration,
then Prometheus exits with exit code `2` before it logs `Starting TSDB ...`.

### AC-4: Missing Rule Files Are Tolerated

Given a `rule_files` entry such as `rules/*.yml` that matches no files,
when I start Prometheus with this configuration,
then Prometheus starts the TSDB and logs `Server is ready to receive web requests.`

### AC-5: Valid Rule Files Load Normally

Given rule files that contain only valid recording and alerting rules,
when I start Prometheus with this configuration,
then Prometheus starts the TSDB, becomes ready,
and the `/api/v1/rules` endpoint lists the rules from these files.

### AC-6: One Invalid File Among Several

Given two `rule_files` patterns, where the first matches only valid files
and the second matches one invalid file,
when I start Prometheus with this configuration,
then Prometheus exits with exit code `2` before it logs `Starting TSDB ...`,
and the error log entry names the invalid file.

### AC-7: Runtime Reload Is Unchanged

Given a running Prometheus with valid rule files,
when I replace a rule file with invalid content and send `SIGHUP`,
then Prometheus logs the reload error and keeps running with the previously
loaded rules, as it did before the change.

### AC-8: Tests Pass

Given the change is complete,
when I run the Go tests for every package you changed,
then the new tests for valid and invalid rule files pass,
and all existing tests pass.

## Definition of Done

- All acceptance criteria are met and you have checked AC-1 to AC-7 manually
  with a locally built Prometheus binary.
- The new code follows the conventions of the surrounding code.
- `make lint` reports no new findings for the changed code, or you have
  documented why a finding is acceptable.

## Reference Solution

The upstream fix is [PR #16601](https://github.com/prometheus/prometheus/pull/16601),
merged as commit `44e27a876`. Compare your solution with it only after you
have finished the kata.
