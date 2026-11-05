# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** with a single root stack template, `cloudformation/template.yaml`. The template creates one AWS Systems Manager (SSM) Parameter Store entry whose name, value, type, description, tier and policies are driven by its parameters.

**Key characteristics:**

- One root template (`cloudformation/template.yaml`) with no nested stacks
- Parameter name built from `ProjectName`, `Environment`, `SSMParamName` and an optional `CiSuffix`
- Advanced-tier policies applied only when `Tier` is `Advanced` and `Policies` is non-empty
- Automated semantic versioning and releases
- AWS OIDC authentication for CI/CD deployments

## Project Structure

```text
cloudformation/
├── template.yaml                  # Root stack: SSM parameter, parameters, conditions, outputs
├── parameters.json                # Parameter values used for deployment
└── stack-config.json              # Stack name and template/parameter file paths for CI

.github/workflows/
├── ci.yaml                        # Validates and deploys templates on changes under cloudformation/
├── release.yaml                   # Semantic release on push to main
├── create-branch.yaml             # Auto-create feature branches from issues
├── claude.yaml                    # Claude workflow
├── claude-code-review.yaml        # Claude code review workflow
├── notify.yaml                    # Notification workflow
└── setup-environments.yaml        # Environment setup workflow

scripts/plugins/
├── release.config.js              # Semantic-release configuration
└── (other release plugins)        # Commit analysis, notes generation, publishing

.claude/                           # Claude Code workspace settings and skills
.devcontainer/devcontainer.json    # Dev container setup (Node.js 20)

package.json                       # Dependencies: semantic-release, commitizen
README.md                          # Template documentation and usage
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat` and `fix` trigger releases.

## Key Architecture Concepts

### Parameter Path

The created parameter's name is:

```text
/{ProjectName}/{Environment}/{SSMParamName}
/{ProjectName}/{Environment}/{SSMParamName}-{CiSuffix}     # when CiSuffix is non-empty
```

The `Name` property uses `Fn::If` on the `HasCiSuffix` condition to choose between the two forms.

### Parameter Types and Tiers

- `ParamType` accepts `String` or `StringList`; the resource's `Type` is set from it.
- `Tier` accepts `Standard`, `Advanced` or `Intelligent-Tiering`, and defaults to `Standard`.
- `Policies` is set on the resource only when `HasAdvancedPolicies` is true (`Tier` is `Advanced` and `Policies` is non-empty). Otherwise the property is omitted with `AWS::NoValue`.

### Encryption

No KMS key parameter exists. `AWS::SSM::Parameter` cannot create `SecureString` parameters, so the template does not offer customer-managed KMS encryption. The `KmsKey` parameter was removed for this reason.

## Key Files to Understand

### `cloudformation/template.yaml`

**Purpose:** Creates one `AWS::SSM::Parameter` and exposes its path and name as outputs.

**Parameters:** `ProjectName`, `Environment`, `SSMParamName`, `ParamType`, `SSMParamValue`, `SSMParamDescription`, `CiSuffix`, `Tier`, `Policies`.

**Conditions:** `HasCiSuffix`, `IsAdvancedTier`, `HasPolicies`, `HasAdvancedPolicies`.

**Outputs:** `ParameterPathPrefix`, `BucketBaseNameParameterName`.

**Resource metadata:** `SSMParameter` has a `com.aws.cloudformation.Context` block that records its purpose and the requirement that the name be unique within the account and region.

**Known issue:** the default `Policies` value includes an `Expiration` timestamp in the past (`2020-05-13`). With `Tier=Advanced` and that default, the deploy will likely fail or the parameter will expire immediately.

### `cloudformation/parameters.json`

Parameter values for deployment. Keys must match parameter names in `template.yaml`; CloudFormation rejects values for undeclared parameters.

### `cloudformation/stack-config.json`

Stack name, template path and parameter file path used by CI.

### `.github/workflows/ci.yaml`

**Triggered on:**

- Manual workflow_dispatch
- Pull requests
- Pushes to `feature/**` and `bug/**` branches

**Path filter:** runs only on changes under `cloudformation/**`.

### `.github/workflows/release.yaml`

**Triggered:** On push to main

**Release rules:**

- `feat:` → MINOR bump (0.1.0 → 0.2.0)
- `fix:` → PATCH bump (0.1.0 → 0.1.1)
- `BREAKING CHANGE:` → MAJOR bump (0.1.0 → 1.0.0)
- Other commits → no release

## Testing & Validation

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
cfn-lint cloudformation/template.yaml
```

**Manual stack deployment:**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-aws-ssm-parameter-stack \
  --parameter-overrides file://cloudformation/parameters.json \
  --region us-east-1
```

## Conventional Commits & Release Flow

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying the Template

1. **Edit `cloudformation/template.yaml`**
2. **Update `cloudformation/parameters.json`** when parameters are added, renamed or removed. Remove keys for parameters the template no longer declares.
3. **Update `README.md`** so the parameter table and examples match the template.
4. **Validate** with `aws cloudformation validate-template` and `cfn-lint`.
5. **Create a PR** with a conventional commit message.
6. **Merge to main** → release workflow creates the version tag and GitHub release.

## Current Branch

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
