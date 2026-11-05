# CloudFormation Nested Stack Template : SSM Parameter

<!-- Row 1: Status - Most Important -->
[![Release](https://img.shields.io/github/v/release/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter?label=Release)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter/releases)&nbsp;[![Release Workflow](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter/actions/workflows/release.yaml)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/badge/Languages-Python%20%7C%20YAML-blue)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ssm-parameter/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/6bd1f947328c829c13d86e991d2b5f76/raw/cfn-nested-aws-ssm-parameter.json)](https://gist.github.com/subhamay-bhattacharyya/6bd1f947328c829c13d86e991d2b5f76)

---

A CloudFormation template that creates an AWS Systems Manager (SSM) Parameter Store entry, with its name built from a project, environment and optional CI suffix.

## Overview

`cloudformation/template.yaml` is a root stack template. It creates one `AWS::SSM::Parameter` whose name follows a fixed path, and whose value, type, description, tier and policies come from its input parameters.

## Repository Layout

```text
cloudformation/
├── template.yaml          # Template: SSM parameter, parameters, conditions, outputs
├── parameters.json        # Parameter values used for deployment
└── stack-config.json      # Stack name and template/parameter file paths for CI
```

## Parameters

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `ProjectName` | String | `cfn-nested` | Top-level path segment. Lowercase letters, numbers, hyphens; max 20 characters. |
| `Environment` | String | `devl` | Second path segment. Lowercase letters, numbers, hyphens. |
| `SSMParamName` | String | `test_name` | Parameter name segment. Lowercase letters, numbers, underscores; max 20 characters. |
| `ParamType` | String | `String` | `String` or `StringList`. `StringList` values must be comma-separated. |
| `SSMParamValue` | String | `ssm_param` | Value stored in the parameter. 1–4096 characters. |
| `SSMParamDescription` | String | `SSM parameter storing the bucket base name` | Description attached to the parameter. Max 1024 characters. |
| `CiSuffix` | String | `""` | Optional. Appended to the name when non-empty. |
| `Tier` | String | `Standard` | `Standard`, `Advanced` or `Intelligent-Tiering`. |
| `Policies` | String | Expiration, ExpirationNotification and NoChangeNotification JSON (see below) | JSON array of SSM policies. Applied only when `Tier` is `Advanced` and the value is non-empty. |

### Default `Policies` value

```json
[{"Type":"Expiration","Version":"1.0","Attributes":{"Timestamp":"2020-05-13T00:00:00.000Z"}},{"Type":"ExpirationNotification","Version":"1.0","Attributes":{"Before":"5","Unit":"Days"}},{"Type":"NoChangeNotification","Version":"1.0","Attributes":{"After":"60","Unit":"Days"}}]
```

> **Warning:** the `Expiration` timestamp above is in the past. SSM expects a future expiration, so deploying with `Tier=Advanced` using this default is likely to fail or create a parameter that expires immediately. Set a future timestamp before using `Advanced`.

## Parameter Name

The parameter name is built from the parameters above:

```text
/{ProjectName}/{Environment}/{SSMParamName}
/{ProjectName}/{Environment}/{SSMParamName}-{CiSuffix}     # when CiSuffix is non-empty
```

Example: `/cfn-nested/devl/bucket_name`

## Outputs

- `ParameterPathPrefix` — `/{ProjectName}/{Environment}`
- `BucketBaseNameParameterName` — name of the created SSM parameter

## Encryption

The template does not support a customer-managed KMS key. CloudFormation's `AWS::SSM::Parameter` resource creates only `String` and `StringList` parameters, and it cannot create `SecureString` parameters. SSM still encrypts parameter values at rest, but with its default handling rather than a key you specify.

## Usage

### Validate

```bash
aws cloudformation validate-template \
  --template-body file://cloudformation/template.yaml
```

### Deploy

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-aws-ssm-parameter-stack \
  --parameter-overrides file://cloudformation/parameters.json \
  --region us-east-1
```

Values in `cloudformation/parameters.json` override the defaults. Parameters left out of that file use their template defaults.

### Lint

```bash
cfn-lint cloudformation/template.yaml
```

## CI

`.github/workflows/ci.yaml` runs when files under `cloudformation/` change. It reads `cloudformation/stack-config.json` for the stack name, template path and parameter file.

## Best Practices Implemented

- ✅ Parameter names unique per project, environment and optional CI suffix
- ✅ Input validation with allowed patterns and allowed values
- ✅ Optional CI suffix support for unique deployments
- ✅ Advanced-tier policies applied only when requested
- ✅ Resource metadata documenting purpose and constraints

## License

MIT
