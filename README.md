# .NET Repo Template

A GitHub repository template for full-stack .NET + Node.js projects with CI, Docker builds, GitOps, and Claude Code integration pre-configured.

## Creating a New Project From This Template

Run the following command to scaffold a new project:

```
dotnet new web-template -n YourProjectName --appName your-project-name --force
```

After scaffolding, push the generated code to a feature branch and raise a PR into `dev`. Do not raise PRs directly into `main` — the branch protection rules will block it.

## Workflow Files

This template includes four pre-configured GitHub Actions workflows. **Do not modify or delete these** — they are correct as checked in and serve as the default for every project created from this template.

| File | Purpose |
|------|---------|
| `ci.yml` | Builds backend (.NET) and frontend (Node.js) on every pull request |
| `check-source-branch.yml` | Enforces that PRs into `main` must come from `dev` |
| `docker-build-push.yml` | Builds and pushes ARM64 Docker images to Oracle Container Registry (OCIR), then auto-bumps the Helm chart version via GitOps |
| `claude.yml` | Enables `@claude` mentions in issues and PRs to trigger Claude Code |

### Important: Claude Cannot Push Workflow Files

GitHub blocks the default Actions token from writing to `.github/workflows/`. This means if Claude is used to scaffold or modify a project, it **cannot push changes to workflow files**. This is intentional GitHub security behaviour, not a bug.

**What to do:** Let Claude push all other project files (source code, configs, Dockerfiles, Helm charts, etc.) and leave the workflow files untouched. The workflows in this template are already correct and will work without modification.

## Branch Protection Rules

Apply **two** rulesets in **Settings → Rules → Rulesets → New ruleset**. They have to be two,
not one: rulesets stack, and stacking only ever *adds* restrictions, so a single ruleset covering
both branches cannot require a status check on `main` and not on `dev`. Requiring it on both is
the trap — see [Why two rulesets](#why-two-rulesets) below.

### 1. `Protected Branches` — `main` and `dev`

```json
{
  "name": "Protected Branches",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "exclude": [],
      "include": [
        "refs/heads/main",
        "refs/heads/dev"
      ]
    }
  },
  "rules": [
    {
      "type": "deletion"
    },
    {
      "type": "non_fast_forward"
    },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 0,
        "dismiss_stale_reviews_on_push": false,
        "required_reviewers": [],
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false,
        "allowed_merge_methods": [
          "rebase",
          "merge"
        ]
      }
    }
  ],
  "bypass_actors": [
    {
      "actor_id": 5,
      "actor_type": "RepositoryRole",
      "bypass_mode": "always"
    },
    {
      "actor_id": 4070856,
      "actor_type": "Integration",
      "bypass_mode": "always"
    }
  ]
}
```

### 2. `Main promotion gate` — `main` only

```json
{
  "name": "Main promotion gate",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "exclude": [],
      "include": [
        "refs/heads/main"
      ]
    }
  },
  "rules": [
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": false,
        "do_not_enforce_on_create": false,
        "required_status_checks": [
          {
            "context": "verify-branch"
          }
        ]
      }
    }
  ]
}
```

Together these enforce:
- `main` and `dev` cannot be deleted or force-pushed
- All changes to either branch go through a pull request
- PRs into `main` must pass the `verify-branch` status check, which is what enforces that they
  come from `dev` (`check-source-branch.yml`)

### Why two rulesets

`check-source-branch.yml` only triggers `on: pull_request: branches: [main]`. It never runs for a
PR into `dev`, so it never posts a `verify-branch` status there.

If a single ruleset requires the `verify-branch` context on `main` **and** `dev`, every
`feature → dev` pull request waits forever on a check that will never report, and the only way to
merge is an admin override. That is not a visible failure — the PR simply sits at "Expected —
waiting for status to be reported" — so it reads as a flaky CI rather than a misconfiguration, and
overriding it becomes routine.

Splitting the required check onto a `main`-only ruleset removes the dead wait without weakening
anything: `main` still cannot be reached except by a PR from `dev`, and `dev` still cannot be
deleted, force-pushed, or written to outside a PR.

## Required Secrets and Variables

For the Docker build workflow to function, configure the following in **Settings → Secrets and variables**:

| Type | Name | Value |
|------|------|-------|
| Variable | `OCIR_REGISTRY` | e.g. `lhr.ocir.io` |
| Variable | `OCIR_NAMESPACE` | Your OCI tenancy namespace |
| Secret | `OCIR_USERNAME` | e.g. `tenancy/oracleidentitycloudservice/you@email.com` |
| Secret | `OCIR_AUTH_TOKEN` | OCI auth token (not your account password) |
| Secret | `CLAUDE_CODE_OAUTH_TOKEN` | OAuth token for Claude Code GitHub Actions |

The `GIT_OPS_APP_ID` variable and `GITOPS_APP_PRIVATE_KEY` secret are pulled from org-level settings and do not need to be set per-repo.
