---
name: terraform-iac
description: Create, modify, plan, apply, and destroy AWS and Azure infrastructure using Terraform Cloud. Use when asked to provision cloud resources, manage IaC, write Terraform configs, run terraform plan/apply/destroy, manage workspaces, or maintain infrastructure state. Triggers on phrases like "create an S3 bucket", "provision a VNet", "deploy infrastructure", "terraform plan", "apply infra", "destroy resources", "show current state", "what's deployed".
---

# Terraform IaC Agent

Manages AWS and Azure infrastructure via Terraform Cloud (TFC). All state lives in TFC — no local state files.

## Configuration

Required env vars (set once in OpenClaw config):

- `TFC_TOKEN` — Terraform Cloud **user API token** or **team API token** (not an organization token — org tokens lack permissions for plan/apply operations). Generate at: User Settings → Tokens, or Organization → Teams → Team API Token.
- `TFC_ORG` — Terraform Cloud organization name
- `TFC_WORKSPACE_AWS` — workspace name for AWS resources (e.g. `prod-aws`)
- `TFC_WORKSPACE_AZURE` — workspace name for Azure resources (e.g. `prod-azure`)

AWS provider credentials set as TFC workspace variables:

- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`

Azure provider credentials set as TFC workspace variables:

- `ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_SUBSCRIPTION_ID`, `ARM_TENANT_ID`

## Workflow

```
User request → Generate/modify .tf files → Run plan → Show output → Wait for approval → Apply
```

**Always show plan and wait for explicit approval before applying.**

### Step 1: Understand the request

Identify:

- Cloud target: AWS, Azure, or both
- Resource type and configuration
- Whether this is create / modify / destroy

### Step 2: Generate Terraform config

Use `{baseDir}/scripts/tfc_client.py generate` with the appropriate resource type:

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource s3 --name my-bucket \
  --workspace prod-aws --dir /tmp/tf-aws
```

Supported resources: `s3`, `vpc`, `ec2`, `sg`, `lambda`, `iam-user`, `iam-role`, `budget`, `cloudtrail`, `cloudwatch`, `efs`, `landing-zone`, `api-gateway`, `rg`

Interactive wizards (vpc, ec2, sg, lambda, iam-user, budget, cloudtrail, cloudwatch, efs, landing-zone, api-gateway) will prompt for configuration with design guidance.

**API Gateway:** The `api-gateway` wizard creates a REST API with usage plan throttling (100 TPS default, burst 50), API key required on all methods, client certificate attached to the stage, CloudWatch execution logging with a dedicated log group, CORS preflight (OPTIONS method with Access-Control-Allow-\* headers), optional WAF WebACL (rate limiting + IP reputation + known bad inputs), and developer portal metadata (private/public/allow-all). A MOCK integration is generated as a placeholder — replace it with your Lambda or HTTP backend before deploying. Plans from the `prod-aws` workspace; does not auto-apply.

**API Gateway non-interactive usage (preferred):**

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource api-gateway --name my-api --workspace prod-aws
```

**API Gateway with custom domain:**

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource api-gateway --name my-api --workspace prod-aws \
  --domain api.example.com --hosted-zone-id Z1234567890ABC
```

When `--domain` and `--hosted-zone-id` are provided, the generator adds an ACM certificate (DNS-validated), Route53 validation records, API Gateway custom domain name (REGIONAL), base path mapping, and a Route53 A-record alias. The certificate is validated automatically via DNS.

**API Gateway with WAF:**

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource api-gateway --name my-api --workspace prod-aws --waf
```

When `--waf` is provided, attaches an AWS WAFv2 WebACL with: rate limiting (2000 req/5min per IP default), AWS Managed IP Reputation List, and AWS Managed Known Bad Inputs rule group. The WAF is associated with the API Gateway stage.

**API Gateway with Lambda backend (creates function if not exists):**

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource api-gateway --name my-api --workspace prod-aws \
  --backend lambda --lambda-runtime python3.12
```

When `--backend lambda` is provided, the generator creates a complete Lambda function (IAM role, zip archive, function resource, API Gateway permission) and wires it as an `AWS_PROXY` integration. The Lambda source code is written to `<dir>/lambda/` with a starter handler. Supported runtimes: `python3.12`, `python3.11`, `nodejs20.x`, `nodejs18.x`.

**API Gateway with HTTP proxy backend:**

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource api-gateway --name my-api --workspace prod-aws \
  --backend http --http-endpoint https://backend.internal/api
```

When `--backend http` is provided, creates an `HTTP_PROXY` integration forwarding all methods to the specified endpoint URL.

**Full example (Lambda + domain + WAF):**

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource api-gateway --name orders --workspace prod-aws \
  --backend lambda --lambda-runtime python3.12 \
  --domain api.example.com --hosted-zone-id Z1234567890ABC --waf
```

**API Gateway with Cognito authorizer:**

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource api-gateway --name my-api --workspace prod-aws \
  --backend lambda --authorizer cognito \
  --cognito-arns "arn:aws:cognito-idp:us-east-1:123456789012:userpool/us-east-1_ABC"
```

When `--authorizer cognito` is provided, creates a `COGNITO_USER_POOLS` authorizer. Clients must send a valid Cognito JWT in the `Authorization` header. Supports comma-separated ARNs for multiple user pools.

**API Gateway with Lambda authorizer (custom token validation):**

```bash
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource api-gateway --name my-api --workspace prod-aws \
  --backend lambda --authorizer lambda
```

When `--authorizer lambda` is provided, creates a TOKEN-based Lambda authorizer with a starter function at `<dir>/authorizer/index.mjs`. The authorizer validates the `Authorization` header and returns an IAM policy. Replace the placeholder validation logic with JWT verification or your own auth scheme. Results are cached for 300s.

When `--name` is provided and `--dir` is omitted, the config is auto-committed to the Git repo at `$TFC_GIT_REPO_DIR/prod-aws/api-gateway-<name>/main.tf` and pushed. Always omit `--dir` for api-gateway so the generated config is persisted in Git.

**VPC CIDR requirement:** The VPC and landing-zone wizards derive /24 subnets from the provided CIDR using proper network arithmetic. The CIDR must be large enough to fit the required subnets (e.g. a /16 or /21 works; a /24 only yields one subnet and will be rejected if multiple AZs are requested).

**Go Lambda:** When the Go runtime is selected, the wizard cross-compiles the function to a Linux/amd64 `bootstrap` binary (requires the Go toolchain on PATH). If Go is not installed, the source is packaged instead with a warning to compile manually before deploying.

See `{baseDir}/references/aws.md` and `{baseDir}/references/azure.md` for provider patterns.

### Step 3: Run plan

```bash
python3 {baseDir}/scripts/tfc_client.py plan --dir /tmp/tf-aws
```

Show the plan output to the user. Summarize what will be created/changed/destroyed.

### Step 4: Wait for approval

Ask: **"The plan looks like X. Shall I apply? (yes/no)"**

Do NOT apply without explicit confirmation.

### Step 5: Apply or abort

```bash
# Apply
python3 {baseDir}/scripts/tfc_client.py apply --dir /tmp/tf-aws

# Destroy (requires separate confirmation)
python3 {baseDir}/scripts/tfc_client.py destroy --confirm DESTROY --dir /tmp/tf-aws
```

### Step 6: Show outputs

After apply, fetch and display outputs:

```bash
python3 {baseDir}/scripts/tfc_client.py outputs --workspace prod-aws
```

## State Queries

To answer "what's currently deployed":

```bash
python3 {baseDir}/scripts/tfc_client.py state --workspace prod-aws
```

## Workspace Management

```bash
python3 {baseDir}/scripts/tfc_client.py list-workspaces
```

## Rules

- Never apply without showing plan first and getting explicit approval
- Never store credentials in `.tf` files — always use TFC workspace variables
- Destroy requires a second explicit confirmation: "Type DESTROY to confirm"
- Keep AWS and Azure in separate TFC workspaces
- Always pin provider versions in `required_providers`

## Git Versioning (Optional)

By default, generated configs live in the `--dir` you specify (e.g. `/tmp/tf-aws`). To persist configs in a Git repo with full history, enable the feature toggle:

### Setup

Add these env vars to your OpenClaw config (`~/.clawdbot/openclaw.json` under `env.vars`):

```
TFC_GIT_ENABLED=true
TFC_GIT_REPO_DIR=~/terraform-iac
TFC_GIT_REPO_URL=https://github.com/<user>/terraform-iac.git
```

- `TFC_GIT_ENABLED` — `true` to activate, omit or `false` to keep default behavior
- `TFC_GIT_REPO_DIR` — local clone path (default: `~/terraform-iac`)
- `TFC_GIT_REPO_URL` — GitHub remote URL (with or without `.git` suffix)

The repo is auto-cloned on first generate if it doesn't exist locally.

### Repo Layout

```
terraform-iac/
├── .gitignore          # auto-generated, excludes .terraform/, tfplan, state, secrets
├── prod-aws/
│   ├── s3-my-bucket/
│   │   └── main.tf
│   ├── vpc-main/
│   │   └── main.tf
│   └── ec2-web-server/
│       └── main.tf
└── prod-azure/
    └── rg-mygroup/
        └── main.tf
```

### Workflow with Git

When enabled, `--dir` becomes optional for `generate` — the path is auto-resolved:

```bash
# --dir is auto-resolved to ~/terraform-iac/prod-aws/s3-my-bucket/
python3 {baseDir}/scripts/tfc_client.py generate \
  --resource s3 --name my-bucket --workspace prod-aws
```

Automatic Git commits happen after:

- `generate` — commits the new/updated `.tf` files
- `plan` — commits any config changes made during plan
- `apply` — commits post-apply state

All commits are pushed to the remote automatically if configured.

### What is NOT stored in Git

- `.terraform/` directory (provider binaries)
- `tfplan` files
- `*.tfstate` / `*.tfstate.*` files (state lives in TFC)
- `*.tfvars` / `*.tfvars.json` (secrets)
- `.terraformrc` / `terraform.rc` (credentials)
