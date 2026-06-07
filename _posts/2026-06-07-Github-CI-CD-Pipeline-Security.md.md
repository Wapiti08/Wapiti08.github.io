---
layout: post
title: "Github CI/CD Pipeline Security"
date: 2026-06-07
tags: [supply chain security, CI/CD pipeline, Github Actions]
---

## Introduction

CI/CD is no longer only a productivity layer. In a modern software supply chain, GitHub Actions can read source code, build artifacts, publish packages, deploy infrastructure, access cloud credentials, and comment back into pull requests. That makes the pipeline part of the production attack surface.

These notes summarize practical lessons from recent GitHub Actions security guidance and incidents, including supply-chain attacks against actions and packages, artifact attestation guidance from GitHub, runtime hardening recommendations from StepSecurity, and Microsoft's 2026 research on agentic CI/CD workflows.

The main assumption is simple:

> If an attacker can influence what runs in CI, what credentials CI can access, or what artifact CI publishes, then the attacker may be able to compromise the software supply chain.

## Threat Model

A secure CI/CD design starts by deciding what cannot be trusted.

In GitHub Actions, the risky inputs usually include:

- Pull request code from forks
- Issue titles, issue bodies, comments, labels, branch names, and commit messages
- Third-party GitHub Actions
- Package manager install scripts and transitive dependencies
- Build artifacts uploaded by earlier jobs
- Self-hosted runner state from previous jobs
- AI-agent prompts or instructions embedded in GitHub content

The sensitive assets usually include:

- `GITHUB_TOKEN`
- Cloud credentials
- Package registry tokens
- Signing keys
- Release artifacts
- Deployment environments
- Repository write permissions
- Secrets exposed as environment variables

The defensive goal is to prevent untrusted input from crossing into privileged execution.

## Why GitHub Actions Is a Supply Chain Target

GitHub Actions is attractive to attackers because it sits at the boundary between source code and production output. A compromised workflow can steal secrets, publish malicious packages, modify releases, poison container images, or push code back into the repository.

Recent public incidents show several recurring patterns:

- Attackers compromise a popular action or dependency.
- Pipelines resolve the compromised version automatically.
- The malicious code runs inside trusted CI.
- Secrets are exfiltrated from the runner.
- Stolen credentials are reused to compromise downstream systems such as package registries.

This means CI/CD security should not only ask "does the code pass tests?" It should also ask:

- Who can change the workflow?
- Which code runs with secrets?
- Which actions are allowed to execute?
- Are actions pinned immutably?
- Can the runner reach the internet freely?
- Can the build artifact be traced back to a trusted workflow?

## Baseline Repository and Organization Controls

Start with GitHub's built-in controls before writing complicated workflow logic.

### 1. Set Default Workflow Permissions to Read-Only

The default `GITHUB_TOKEN` should not have write access unless a workflow explicitly needs it.

At the organization or repository level:

- Set workflow token permissions to read-only.
- Disable broad write permissions by default.
- Avoid allowing GitHub Actions to create or approve pull requests unless there is a strong reason.

Inside workflow files, declare permissions explicitly:

```yaml
permissions: {}
```

Then grant only the permissions each job needs:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

For release jobs, make the permission increase visible and isolated:

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
      id-token: write
```

The important habit is to treat permissions as part of the workflow API, not as an implementation detail.

### 2. Protect Main and Release Branches

Branch protection and rulesets reduce the chance that an attacker can merge malicious code into trusted branches.

Useful controls include:

- Require pull request reviews before merge.
- Require status checks to pass.
- Dismiss stale approvals when new commits are pushed.
- Require approval from someone other than the last pusher.
- Restrict who can push to release branches.
- Require signed commits or verified provenance for high-value repositories.

These controls matter because CI/CD often gives more privilege to workflows running from `main`, release branches, or protected tags.

### 3. Govern Which Actions Can Run

Third-party actions are dependencies. They should be governed like packages.

At the organization level:

- Prefer GitHub-owned and verified actions.
- Maintain an allowlist for third-party actions.
- Block known compromised actions quickly.
- Enforce full-length SHA pinning where possible.

Using a tag is convenient:

```yaml
- uses: actions/checkout@v4
```

But a full commit SHA is more immutable:

```yaml
- uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332
```

The tradeoff is update management. SHA pinning improves integrity, but teams need automation such as Dependabot, Renovate, or a controlled update process to avoid freezing old vulnerable versions forever.

## Secure Workflow Design

Workflow design should keep untrusted code away from privileged execution.

### Avoid Dangerous `pull_request_target` Patterns

`pull_request_target` runs in the context of the base repository. That means it can access higher privileges than a normal fork pull request workflow.

The dangerous pattern is:

```yaml
on:
  pull_request_target:

jobs:
  dangerous:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - run: npm install
      - run: npm test
```

This checks out attacker-controlled code and runs it in a privileged context. If secrets or write permissions are available, the workflow can become a direct secret-exfiltration path.

Safer patterns:

- Use `pull_request` for testing untrusted code.
- Keep `pull_request_target` limited to metadata-only tasks such as labeling or commenting.
- Never run fork code with repository secrets.
- Split workflows so untrusted build/test jobs cannot access deployment credentials.

### Prevent Script Injection

GitHub context values can be attacker-controlled. Examples include issue titles, PR titles, branch names, labels, and comments.

Avoid interpolating untrusted values directly into shell commands:

```yaml
# Unsafe
- run: echo "PR title: ${{ github.event.pull_request.title }}"
```

Prefer passing values through environment variables and quoting carefully:

```yaml
- name: Print PR title
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"
```

For complex workflows, validate inputs before use. A good rule: if a value came from a user, issue, PR, branch name, or comment, treat it as data, not code.

### Avoid Exposing the Full Secrets Context

Do not pass all secrets into a job or step:

```yaml
# Unsafe
env:
  SECRETS: ${{ toJson(secrets) }}
```

Instead, pass only the exact secret needed by the exact step:

```yaml
- name: Publish package
  env:
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
  run: npm publish
```

Also avoid `secrets: inherit` in reusable workflows unless the reusable workflow genuinely needs every inherited secret. Prefer explicit secret contracts:

```yaml
jobs:
  deploy:
    uses: org/repo/.github/workflows/deploy.yml@main
    secrets:
      cloud_role: ${{ secrets.PROD_DEPLOY_ROLE }}
```

## Secrets and Identity

Long-lived CI secrets are one of the highest-risk parts of the pipeline. If a workflow can print, upload, or send them over the network, an attacker only needs one injection point.

### Prefer OIDC Over Static Cloud Keys

OpenID Connect lets GitHub Actions request short-lived cloud credentials from AWS, Azure, GCP, and other providers. Instead of storing a permanent cloud key in GitHub, the workflow receives a temporary token bound to repository, branch, workflow, or environment conditions.

This improves supply-chain security because:

- There is no long-lived cloud key to steal from GitHub secrets.
- The cloud provider can restrict which repository and branch may assume a role.
- Tokens expire quickly.
- Access can be scoped to a specific deployment or publishing task.

Typical workflow permission:

```yaml
permissions:
  contents: read
  id-token: write
```

The `id-token: write` permission should be granted only to jobs that actually need federated identity.

### Use Environment Secrets With Required Reviews

GitHub environments are useful for sensitive stages such as production deployment.

For production:

- Store deployment secrets as environment secrets.
- Require manual approval before jobs can access the environment.
- Restrict which branches can deploy.
- Separate staging and production credentials.

This creates a human-controlled gate between untrusted code changes and production credentials.

### Be Careful With Logs and Artifacts

GitHub masks known secrets in logs, but masking is not a complete data-loss prevention system.

Avoid:

- Printing secrets for debugging
- Storing structured JSON blobs as a single secret
- Uploading `.env`, credentials, `.npmrc`, cloud config, or `.git/config`
- Uploading entire workspaces as artifacts

When using `actions/checkout`, consider disabling persisted credentials unless later steps need them:

```yaml
- uses: actions/checkout@v4
  with:
    persist-credentials: false
```

## Runner Security

Runners are execution infrastructure. Treat them like production machines.

### Prefer Ephemeral Runners

GitHub-hosted runners are ephemeral by default. For many repositories, that is safer than maintaining long-lived self-hosted runners.

Self-hosted runners are useful for performance, private networking, or custom hardware, but they introduce persistence risk:

- Malware can remain between jobs.
- Build tools can modify the host.
- Secrets may remain in caches or temporary directories.
- Public pull requests can execute code on infrastructure you own if runner routing is too broad.

For self-hosted runners:

- Use ephemeral runners where possible.
- Isolate runners by repository, environment, and trust level.
- Do not attach public repositories to sensitive self-hosted runners.
- Harden runner images.
- Monitor process activity, file writes, and network connections.
- Clear workspaces and caches between jobs.

### Restrict Network Egress

Many CI attacks end with outbound data transfer. A runner that can connect anywhere can send secrets anywhere.

A stronger model is default-deny egress with allowlists for:

- GitHub APIs
- Required package registries
- Container registries
- Cloud provider APIs
- Internal deployment endpoints

For GitHub-hosted runners, full network control can be harder. At minimum, monitor outbound connections and investigate unexpected domains during build and release jobs.

## Artifact Attestations and SLSA

Securing the workflow is only half the problem. Consumers also need to know where an artifact came from.

Artifact attestations provide provenance: a signed statement about how an artifact was built, usually including the repository, workflow, commit, and build identity.

GitHub's artifact attestation guidance shows how reusable workflows and attestations can help projects move toward SLSA v1.0 Build Level 3.

For a workflow that generates attestations, both the caller and reusable workflow need permissions like:

```yaml
permissions:
  attestations: write
  contents: read
  id-token: write
```

A simplified release flow:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: read
  id-token: write
  attestations: write

jobs:
  build:
    uses: org/security-workflows/.github/workflows/build-and-attest.yml@main
```

Then verify the artifact before consuming or deploying it:

```bash
gh attestation verify ./dist/app.tar.gz --repo owner/repo
```

Attestation is not a replacement for secure CI. It is evidence that lets downstream systems reject artifacts that were not built by an approved workflow.

## Reusable Workflows as Security Boundaries

Reusable workflows help centralize security-sensitive tasks.

Good candidates:

- Production builds
- Container image publishing
- Package releases
- Deployment
- Artifact attestation
- Security scanning

Benefits:

- Security teams can maintain one hardened workflow.
- Application teams use a standard interface.
- Permissions and secrets are easier to audit.
- Attestation policies can require artifacts to come from approved workflows.

However, reusable workflows should be pinned and reviewed just like actions. They are part of the trusted computing base.

## Agentic CI/CD Security

AI coding agents inside GitHub workflows introduce a new version of an old problem: untrusted input influencing privileged automation.

Microsoft's June 2026 research on the Claude Code GitHub Action showed that AI agents processing untrusted GitHub content can be steered by prompt injection. In their case study, attacker-controlled issue or PR content could influence an agent with repository tools, and a file-read path exposed sensitive runner environment data before the vendor mitigation.

The lesson is broader than one tool:

> Treat AI agents in CI as privileged automation that can be prompt-injected.

Risky pattern:

- A workflow runs on issues, comments, or pull requests from untrusted users.
- The agent reads raw GitHub content.
- The agent has file read/write tools, shell tools, PR creation tools, or network access.
- Secrets are present in the same job.

Safer design:

- Do not expose secrets to agent jobs that process untrusted text.
- Use read-only repository permissions by default.
- Require human approval before agent-created changes are merged.
- Disable shell tools unless required.
- Restrict file access to specific paths.
- Restrict network egress.
- Log agent tool calls for review.
- Treat hidden markdown, HTML comments, and issue templates as untrusted prompt input.

An AI workflow should have the same separation as any other CI workflow: untrusted interpretation in one low-privilege job, privileged action in a separate reviewed job.

## A Practical Hardening Checklist

### Organization Level

- Set default `GITHUB_TOKEN` permissions to read-only.
- Restrict allowed third-party actions.
- Enforce SHA pinning where possible.
- Use branch protection and rulesets.
- Restrict self-hosted runners to specific repositories.
- Inventory workflows, secrets, actions, and runners.

### Workflow Level

- Add `permissions: {}` at the top.
- Grant permissions per job.
- Avoid running untrusted code in `pull_request_target`.
- Pass secrets only to the steps that need them.
- Avoid `secrets: inherit` unless necessary.
- Use OIDC instead of long-lived cloud secrets.
- Disable `persist-credentials` for checkout when possible.
- Pin actions and reusable workflows.
- Validate or safely quote untrusted inputs.

### Runtime Level

- Prefer ephemeral runners.
- Isolate self-hosted runners by trust level.
- Restrict outbound network access.
- Monitor runner behavior.
- Scan logs and artifacts for accidental secrets.
- Keep runner images minimal and patched.

### Release Level

- Build releases from protected refs.
- Generate artifact attestations.
- Verify attestations before deployment.
- Use reusable workflows for release and deployment.
- Keep signing and publishing credentials separate from test workflows.

### Agentic Workflow Level

- Treat issue bodies, comments, PR descriptions, and hidden markdown as hostile input.
- Keep AI agent jobs secret-free by default.
- Limit tools available to agents.
- Require human review for agent-created code or PRs.
- Monitor tool calls and outbound traffic.

## Example Secure Workflow Skeleton

This example separates untrusted pull request testing from privileged release behavior.

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main

permissions: {}

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test

  build:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      id-token: write
      attestations: write
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - name: Build
        run: npm run build
      - name: Generate artifact attestation
        uses: actions/attest-build-provenance@v2
        with:
          subject-path: ./dist/**
```

This is not complete for every project, but the shape is important:

- Pull requests run with read-only access.
- Release-like behavior only runs from `main`.
- Attestation requires explicit permissions.
- Checkout does not persist credentials unless needed.
- Secrets are absent from test jobs.

## Mental Model

The most useful mental model is to split CI/CD into trust zones:

1. **Untrusted zone:** fork code, comments, issue content, dependency install scripts, AI prompts.
2. **Build zone:** deterministic build steps with minimal credentials.
3. **Release zone:** protected branches, reviewed workflows, short-lived identity, artifact attestations.
4. **Deploy zone:** environment approvals, cloud-side policy, monitored credentials.

A secure pipeline reduces movement between these zones. When movement is necessary, it adds verification: review, permission checks, provenance, attestation, and runtime monitoring.

## References

- [How to Harden GitHub Actions: An Updated Guide](https://www.wiz.io/blog/github-actions-security-guide)
- [Using artifact attestations and reusable workflows to achieve SLSA v1 Build Level 3](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/increase-security-rating)
- [7 GitHub Actions Security Best Practices](https://www.stepsecurity.io/blog/github-actions-security-best-practices)
- [Securing CI/CD in an agentic world: Claude Code GitHub Action case](https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/)