---
description: Weekly security workflow with multiple stages

on:
  schedule: weekly
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read

imports:
  - ../agents/Frontend-Security-Scanner.md

network: defaults

safe-outputs:
  create-issue:
    title-prefix: "[Security Audit] "
    labels: [security, audit, weekly-report]

engine: copilot
---

# Weekly Security Workflow

## Stage 1: Run Frontend Security Scanner

Invoke the Frontend Security Scanner agent to perform a full security audit.

Agent: `.github/agents/Frontend-Security-Scanner.md`

## Stage 2: Process Results

Take the security scanner's findings and:

1. Create a detailed GitHub issue
2. Mention @pr-team
3. If critical issues found, also create a discussion for immediate attention

## Stage 3: Track Remediation

Check if previous security issues have been resolved.
