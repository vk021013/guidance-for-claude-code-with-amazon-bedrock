# OneLogin Complete Integration Guide

This guide covers every aspect of integrating OneLogin with the Claude Code + Amazon Bedrock solution — from initial OneLogin app configuration through Claude Code CLI, Claude Cowork (Desktop), monitoring, analytics, quota enforcement, and cost attribution.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [OneLogin Application Setup](#3-onelogin-application-setup)
4. [Deploy AWS Infrastructure](#4-deploy-aws-infrastructure)
5. [How Authentication Works](#5-how-authentication-works)
6. [Claude Code CLI](#6-claude-code-cli)
7. [Claude Cowork 3P (Claude Desktop)](#7-claude-cowork-3p-claude-desktop)
8. [Monitoring](#8-monitoring)
9. [Analytics Pipeline](#9-analytics-pipeline)
10. [Quota Monitoring](#10-quota-monitoring)
11. [Cost Attribution](#11-cost-attribution)
12. [Distribution to End Users](#12-distribution-to-end-users)
13. [Credential Storage](#13-credential-storage)
14. [Silent Refresh](#14-silent-refresh)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. Overview

This solution deploys a single AWS authentication stack that federates OneLogin identities to Amazon Bedrock. A single deployment covers both:

- **Claude Code** — terminal-based coding tool for developers
- **Claude Cowork** — Claude Desktop for knowledge workers (product managers, analysts, operations)

No Anthropic API keys are distributed to end users. Instead, users authenticate with their corporate OneLogin credentials and receive short-lived AWS temporary credentials scoped to Bedrock access.

### What OneLogin integration provides

| Capability | Details |
|---|---|
| **Authentication** | OAuth 2.0 Authorization Code + PKCE (no client secret needed) |
| **AWS Federation** | Direct STS (`AssumeRoleWithWebIdentity`) or Cognito Identity Pool |
| **Session duration** | 12 hours (Direct STS) or 8 hours (Cognito) |
| **Auto-refresh** | Silent credential refresh using cached OIDC token |
| **User attribution** | Email embedded in STS session name → per-user cost tracking |
| **Quota enforcement** | Per-user token limits enforced at credential issuance |
| **Group-based policies** | OneLogin roles/groups mapped to quota tiers |

### OIDC Endpoints Used

| Endpoint | URL |
|---|---|
| Authorization | `https://amagi-media-labs.onelogin.com/oidc/2/auth` |
| Token exchange | `https://amagi-media-labs.onelogin.com/oidc/2/token` |
| Scopes | `openid profile email` |
| Flow | Authorization Code + PKCE |

---

## 2. Prerequisites

### For administrators (deploying the solution)

- Python 3.10–3.13 + Poetry
- AWS CLI v2 with IAM/CloudFormation permissions
- OneLogin account with administrator access
- Amazon Bedrock activated in target AWS regions

### For end users

- Claude Code installed (for CLI use)
- Claude Desktop installed (for Cowork use)
- Web browser for SSO login (first-time and on token expiry)
- No AWS account, no Python, no credentials to manage

---

## 3. OneLogin Application Setup

### Step 1 — Create an OIDC application

1. Log in to the OneLogin Admin Console: `https://amagi-media-labs.onelogin.com/admin`
2. Go to **Applications → Applications → Add App**
3. Search for **OpenID Connect (OIDC)** and select it
4. Name: `Amazon Bedrock CLI Access`
5. Click **Save**

### Step 2 — Configure the SSO tab

Under the **SSO** tab:

| Setting | Value |
|---|---|
| **Application Type** | `Web` |
| **Authentication Method** | `none` |

> **Only Client ID is required — no client secret.** Setting Authentication Method to `none` enables PKCE-only (public client) mode. The credential provider sends only the `client_id` and a PKCE `code_verifier` during token exchange. No secret is ever stored on user machines.

Under the **Configuration** tab:

| Setting | Value |
|---|---|
| **Redirect URI** | `http://localhost:8400/callback` |
| **Post Logout Redirect URI** | `http://localhost:8400/logout` (optional) |

### Step 3 — Collect required values

After saving, note:

| Parameter | Where to find it | Example |
|---|---|---|
| **OneLogin Domain** | Your org's subdomain | `amagi-media-labs.onelogin.com` |
| **Client ID** | SSO tab → Client ID | `abc123...` |

> No client secret needed.

### Step 4 — Assign users

1. Go to the **Users** tab on the application
2. Assign individual users, or assign OneLogin roles/groups

### Step 5 — (Optional) Add groups claim for quota policies

If you want group-based quota limits (e.g., engineers get more tokens than trial users):

1. Go to the **Parameters** tab on the application
2. Click **+** to add a parameter
3. Set **Field name**: `groups`
4. Set **Value**: the OneLogin role attribute (e.g., `User Roles`)
5. Check **Multi-value parameter**
6. Click **Save**

This adds a `groups` array to the JWT, which the quota system reads to apply group-level policies.

---

## 4. Deploy AWS Infrastructure

### Step 1 — Initialize the profile

```bash
poetry run ccwb init
```

When prompted:

```
Enter your OIDC provider domain: amagi-media-labs.onelogin.com
# Auto-detected as: OneLogin ✓

Enter your Client ID: <your-client-id>

Select federation type:
  > Direct IAM (recommended — 12 hour sessions)
    Cognito Identity Pool (legacy — 8 hour sessions)

Select AWS region: us-east-1
Select Bedrock regions: us-east-1, us-west-2
Enable monitoring: Yes
```

### Step 2 — Deploy the authentication stack

```bash
poetry run ccwb deploy auth
```

This deploys `bedrock-auth-onelogin.yaml`, which creates:

- **IAM OIDC Provider** — registers `https://amagi-media-labs.onelogin.com/oidc/2` as a trusted identity source
- **IAM Role** (`BedrockOneLoginFederatedRole`) — federated role with Bedrock access scoped to allowed regions
- **Bedrock Access Policy** — allows `bedrock:InvokeModel`, `bedrock-runtime:Converse`, `bedrock-runtime:ConverseStream`, and related actions
- **(Cognito mode only)** Cognito Identity Pool with principal tag mapping

### Step 3 — (Optional) Deploy monitoring stack

```bash
poetry run ccwb deploy monitoring
```

### Step 4 — (Optional) Deploy quota monitoring

```bash
poetry run ccwb deploy quota
```

### Step 5 — Build distribution packages

```bash
poetry run ccwb package
```

---

## 5. How Authentication Works

### Full flow (first login)

```
User runs Claude Code
       │
       ▼
credential-process checks cache → cache miss
       │
       ▼
Opens browser → amagi-media-labs.onelogin.com/oidc/2/auth
  (PKCE: code_challenge sent, no client secret)
       │
       ▼
User logs in with OneLogin credentials (MFA if configured)
       │
       ▼
OneLogin redirects to localhost:8400/callback with auth code
       │
       ▼
credential-process exchanges code for id_token + access_token
  POST amagi-media-labs.onelogin.com/oidc/2/token
  body: client_id + code + code_verifier (no secret)
       │
       ▼
[Optional] Quota check — API Gateway validates JWT, checks DynamoDB
       │
       ▼
Direct STS: sts:AssumeRoleWithWebIdentity(id_token, role_arn)
  → session name = user email (e.g., alice@amagi.com)
  → returns 12-hour temporary credentials
       │
       ▼
Credentials cached (keyring or ~/.aws/credentials)
id_token cached for silent refresh
       │
       ▼
AWS credentials printed to stdout → Claude Code uses them
```

### Silent refresh (subsequent credential requests)

When cached AWS credentials expire but the OIDC id_token is still valid:

```
credential-process checks AWS credentials → expired
       │
       ▼
Checks cached id_token → still valid
       │
       ▼
Re-calls AssumeRoleWithWebIdentity with same id_token
  → new 12-hour credentials
  → no browser popup required
```

A browser popup only appears when the id_token itself has also expired.

### Provider detection

The domain `amagi-media-labs.onelogin.com` is auto-detected as `onelogin` by matching `*.onelogin.com`. No manual provider_type configuration is needed.

---

## 6. Claude Code CLI

### What users experience

1. User runs `claude` in the terminal
2. On first use, a browser window opens to OneLogin for SSO login
3. After login, Claude Code starts with Bedrock as the inference backend
4. Subsequent runs use cached credentials — no browser popup until the OIDC token expires

### Configuration written by the installer

The `install.sh` / `install.bat` distributed by `ccwb package` writes:

**`~/.aws/config`:**
```ini
[profile ClaudeCode]
credential_process = ~/claude-code-with-bedrock/credential-process --profile ClaudeCode
```

**`~/.claude/settings.json`:**
```json
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "AWS_PROFILE": "ClaudeCode",
    "AWS_REGION": "us-east-1",
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://<your-alb>.elb.amazonaws.com"
  },
  "otelHeadersHelper": "~/claude-code-with-bedrock/otel-helper"
}
```

### Models available

Cross-region inference profiles are automatically configured:

| Alias | Model |
|---|---|
| `opus` | `us.anthropic.claude-opus-4-7` |
| `sonnet` | `us.anthropic.claude-sonnet-4-6` |
| `haiku` | `us.anthropic.claude-haiku-4-5` |

---

## 7. Claude Cowork 3P (Claude Desktop)

The same credential helper binary and AWS profile used by Claude Code CLI also power Claude Cowork. No additional AWS infrastructure is needed.

### How credential flow works for Cowork

1. Claude Desktop reads `inferenceBedrockProfile` from the MDM policy
2. Passes the profile name to the AWS SDK
3. SDK runs `credential_process = credential-process --profile ClaudeCode`
4. `credential-process` authenticates via OneLogin and returns temporary credentials
5. Claude Desktop signs each Bedrock API call with those credentials

### Generate MDM configuration

```bash
poetry run ccwb cowork generate
```

This outputs to `dist/cowork-3p/`:
- `cowork-3p-config.json` — base MDM configuration
- `cowork-3p.mobileconfig` — macOS MDM profile (deploy via Jamf, Kandji, Mosyle)
- `cowork-3p.reg` — Windows registry file (deploy via Intune or Group Policy)

### MDM configuration keys

```json
{
  "inferenceProvider": "bedrock",
  "inferenceBedrockRegion": "us-east-1",
  "inferenceBedrockProfile": "ClaudeCode",
  "inferenceModels": ["opus", "sonnet", "haiku"]
}
```

If monitoring is enabled, the OTLP endpoint is automatically included:

```json
{
  "inferenceProvider": "bedrock",
  "inferenceBedrockRegion": "us-east-1",
  "inferenceBedrockProfile": "ClaudeCode",
  "inferenceModels": ["opus", "sonnet", "haiku"],
  "otlpEndpoint": "http://<your-alb>.elb.amazonaws.com",
  "otlpProtocol": "http/protobuf"
}
```

### Deploying MDM config

**macOS (via Jamf/Kandji/Mosyle):**
1. Upload `cowork-3p.mobileconfig` to your MDM
2. Scope to target devices/users
3. Deploy — takes effect on next MDM check-in

**Windows (via Intune/Group Policy):**
1. Deploy `cowork-3p.reg` via a script or software deployment policy
2. Or import via Group Policy Preferences → Registry

**Manual testing:**
```bash
# Apply locally for testing (no MDM needed)
# Open Claude Desktop → Help → Troubleshooting → Developer Mode
# → Configure third-party inference → Apply locally
```

### Important: credential storage for Cowork

> **Use keyring mode** when Cowork 3P is in scope. The AWS SDK resolves `~/.aws/credentials` before `credential_process` in `~/.aws/config`. If `~/.aws/credentials` contains an expired stanza for the profile, Cowork fails with `403 InvalidClientTokenId`. Keyring mode never writes to `~/.aws/credentials`, eliminating this race condition.

Set during `ccwb init`:
```
Credential storage: Keyring (recommended for Cowork)
```

### Features available in Cowork with Bedrock

| Feature | Available |
|---|---|
| Projects | ✅ |
| Artifacts | ✅ |
| File upload/export | ✅ |
| MCP servers | ✅ |
| Remote connectors | ✅ |
| Memory | ✅ |
| Chat tab | ❌ (requires Anthropic-hosted inference) |
| Computer Use | ❌ |
| Skills Marketplace | ❌ |

---

## 8. Monitoring

Monitoring is optional but strongly recommended for organizations with more than a handful of users.

### Architecture

```
Claude Code / Claude Cowork
        │  OTLP metrics (HTTP/protobuf)
        ▼
Application Load Balancer
        │
        ▼
ECS Fargate (ADOT Collector)
  - Extracts user identity from JWT headers (via otel-helper)
  - Batches metrics every 60 seconds
        │
        ▼
CloudWatch Metrics + Logs
        │
        ▼
CloudWatch Dashboard (ClaudeCodeMonitoring)
        │
        ▼ (optional)
Kinesis Firehose → S3 → Athena (analytics pipeline)
```

### Deploying monitoring

```bash
poetry run ccwb deploy monitoring
```

This creates:
- VPC with public/private subnets (or use existing VPC)
- ECS Fargate cluster running ADOT Collector
- Application Load Balancer on port 4318
- CloudWatch log groups and dashboards
- DynamoDB for metrics aggregation

### Metrics collected

| Metric | Description |
|---|---|
| `claude_code.token.usage` | Input/output/cache tokens per request |
| `claude_code.cost.usage` | Estimated USD cost (client-side) |
| `claude_code.session.count` | Active sessions |
| `claude_code.active_time.total` | Time actively using Claude Code |
| `claude_code.code_edit_tool.decision` | Code editing decisions |

### User identity in metrics

The `otel-helper` binary reads the cached OneLogin id_token and injects user identity as HTTP headers on each metric request. The ADOT Collector maps these headers to CloudWatch dimensions:

| Dimension | JWT Claim | Example |
|---|---|---|
| `UserEmail` | `email` | `alice@amagi.com` |
| `UserId` | `sub` | `12345678` |
| `UserName` | `name` | `Alice` |
| `department` | `custom:department` | `engineering` |

### CloudWatch Dashboard

The `ClaudeCodeMonitoring` dashboard shows:
- Token consumption by user, model, and type
- Active users and top consumers
- Hourly/daily usage heatmaps
- Prompt cache hit rates and token savings
- Estimated Bedrock costs

---

## 9. Analytics Pipeline

The analytics pipeline extends monitoring with a historical data lake for SQL-based analysis.

### Deploy

```bash
poetry run ccwb deploy analytics
```

Or via CloudFormation directly:
```bash
aws cloudformation deploy \
  --template-file deployment/infrastructure/analytics-pipeline.yaml \
  --stack-name claude-code-analytics \
  --capabilities CAPABILITY_IAM
```

### Architecture

```
CloudWatch Logs
      │
      ▼
Kinesis Data Firehose
      │  (Parquet format, partitioned by year/month/day/hour)
      ▼
S3 Data Lake
  ├─ Standard: 0–90 days
  └─ Glacier: 90+ days (automatic)
      │
      ▼
AWS Athena (SQL queries, no Glue crawlers needed)
```

### Pre-built Athena queries

10 saved queries are auto-created in your workgroup:

| Query | Use case |
|---|---|
| Top Users by Token Usage | Identify power users and track consumption |
| Token Usage by Model and Type | Optimize model selection and cost distribution |
| User Activity Pattern by Hour | Capacity planning, peak usage identification |
| Token Usage by Organization | Org-level billing and chargeback |
| Token Usage by Email Domain | Department/team analysis |
| TPM and RPM Analysis | Rate limit monitoring |
| User Session Analysis | Session duration, intensity, per-session cost |
| Detailed Cost Attribution | Per-user, per-org, per-model cost calculation |
| Peak Usage and Rate Limit Analysis | Proactive monitoring |
| Usage Analysis by Identity Provider | Compares usage across IdPs |

### Sample query — filter for OneLogin users

```sql
SELECT
  user_email,
  SUM(token_count) AS total_tokens,
  COUNT(DISTINCT session_id) AS sessions,
  SUM(token_count) * 15.0 / 1000000 AS estimated_cost_usd
FROM claude_code_metrics
WHERE year >= YEAR(CURRENT_DATE - INTERVAL '30' DAY)
  AND from_unixtime(timestamp/1000) >= CURRENT_TIMESTAMP - INTERVAL '30' DAY
  AND user_email LIKE '%@amagi.com'
GROUP BY user_email
ORDER BY total_tokens DESC
LIMIT 20;
```

---

## 10. Quota Monitoring

Quota monitoring enforces per-user token limits at credential issuance time, before any Bedrock API call is made.

### How it works with OneLogin

1. User authenticates via OneLogin → receives id_token
2. `credential-process` extracts `email` and `groups` claims from the JWT
3. Calls the Quota Check API (`GET /check`) with the id_token in the `Authorization: Bearer` header
4. API Gateway validates the JWT against the OneLogin OIDC provider
5. Lambda checks the user's monthly/daily usage in DynamoDB
6. If over limit → credential issuance is blocked; browser notification shown
7. If under limit → AWS credentials are issued normally

### Deploy quota monitoring

```bash
poetry run ccwb deploy quota
```

### Configure limits

```bash
# Set default limit for all users
poetry run ccwb quota set-default --monthly-limit 225M

# Group-based limits (requires groups claim from OneLogin)
poetry run ccwb quota set-group engineering --monthly-limit 500M
poetry run ccwb quota set-group data-science --monthly-limit 1B

# User-specific override
poetry run ccwb quota set-user alice@amagi.com --monthly-limit 750M
```

### Quota policy precedence

```
User-specific policy  (highest priority)
        │
Group policy (matched against OneLogin roles/groups)
        │
Default policy        (lowest priority, required)
```

### OneLogin groups claim

The `groups` claim in the OneLogin JWT is read as:

```json
{
  "sub": "12345678",
  "email": "alice@amagi.com",
  "groups": ["engineering", "bedrock-power-users"]
}
```

The quota system tries each group in order until a matching policy is found.

To include `cognito:groups`-style claims, you can also map OneLogin roles to custom claims. The system also reads `custom:department` as a `department:<value>` group key.

### Limit types and enforcement

| Limit | Default | Enforcement |
|---|---|---|
| Monthly tokens | 225M | `block` (access denied) |
| Daily tokens | ~8.25M (auto) | `alert` (warn only) |
| Warning threshold | 80% (180M) | Email notification via SNS |
| Critical threshold | 90% (202.5M) | Email notification via SNS |

### What users see when blocked

A terminal message plus a browser notification page with usage progress bars:

```
============================================================
ACCESS BLOCKED - QUOTA EXCEEDED
============================================================

Monthly: 225,000,000 / 225,000,000 tokens (100.0%)

To request an unblock, contact your administrator.
============================================================
```

### Fail mode

| Mode | Behavior on API error |
|---|---|
| `open` (default) | Allow access if quota API is unreachable |
| `closed` | Deny access if quota API is unreachable |

---

## 11. Cost Attribution

### Built-in per-user tracking (no IdP changes needed)

The credential provider embeds the user's email in the STS session name:

```
arn:aws:sts::123456789012:assumed-role/BedrockOneLoginFederatedRole/alice@amagi.com
```

This ARN appears in the `line_item_iam_principal` column of AWS Cost and Usage Report (CUR 2.0), giving you per-user Bedrock costs without any additional configuration.

**To enable:**
1. Open Billing → Data Exports → edit your CUR 2.0 export
2. Enable **"Include caller identity (IAM principal) allocation data"**
3. Query via Athena: filter `line_item_iam_principal` by email

### Optional: Session tags for Cost Explorer

If you need per-user costs in Cost Explorer (not just CUR), configure OneLogin to inject session tags into the id_token.

**OneLogin custom claim configuration:**

Add a custom claim in your OneLogin OIDC app under **Parameters**:

| Field name | Value |
|---|---|
| `https://aws.amazon.com/tags/principal_tags/UserEmail` | User email attribute |
| `https://aws.amazon.com/tags/principal_tags/UserId` | User ID attribute |
| `https://aws.amazon.com/tags/transitive_tag_keys` | `["UserEmail", "UserId"]` |

> **Note:** OneLogin must support JSON object values in OIDC claims for the nested format. If not, use the flattened per-key format shown above.

**The IAM role trust policy** (already included in `bedrock-auth-onelogin.yaml`) includes `sts:TagSession`:

```yaml
Action:
  - 'sts:AssumeRoleWithWebIdentity'
  - 'sts:TagSession'
```

**Activate tags in Cost Allocation:**
1. Billing → Cost Allocation Tags → User-defined tags
2. Activate `UserEmail` and `UserId`
3. Wait up to 24 hours for tags to appear in Cost Explorer

---

## 12. Distribution to End Users

### Package the distribution

```bash
poetry run ccwb package
```

Builds platform-specific executables in `dist/`:

| Platform | File |
|---|---|
| macOS ARM64 | `dist/macos-arm64/` |
| macOS Intel | `dist/macos-x64/` |
| Linux x86_64 | `dist/linux-x64/` |
| Windows x64 | `dist/windows-x64/` |

Each package contains:
- `credential-process` binary (standalone, no Python required)
- `otel-helper` binary
- `config.json` (pre-configured with your OneLogin domain and client ID)
- `install.sh` / `install.bat`
- `claude-settings/settings.json`

### Distribution methods

**Manual sharing:**
Zip the platform folder and share via email or internal file sharing. No extra infrastructure needed.

**Presigned S3 URLs:**
```bash
poetry run ccwb deploy distribution --type presigned-s3
poetry run ccwb distribute generate-links
```
Generates time-limited download links per platform.

**Self-service landing page:**
```bash
poetry run ccwb deploy distribution --type landing-page --idp onelogin
```
Deploys a web portal where users authenticate with OneLogin and download their platform's package automatically.

---

## 13. Credential Storage

Two modes are available. **Keyring is recommended**, especially when Claude Cowork 3P is deployed.

### Keyring mode

Stores AWS credentials in the OS secure store (macOS Keychain, Windows Credential Manager, Linux Secret Service).

- Credentials never written to `~/.aws/credentials`
- Required for Cowork 3P to avoid the boto3 credentials file resolution order issue
- On Windows, the session token is split across 4 entries due to the 2560-byte limit

### Session mode

Stores credentials in `~/.aws/credentials` under the `[ClaudeCode]` profile section with an `x-expiration` custom key.

- Simpler for Claude Code CLI-only deployments
- Not recommended when Cowork 3P is also deployed

Set during `ccwb init` or in config:
```json
{
  "credential_storage": "keyring"
}
```

---

## 14. Silent Refresh

Silent refresh minimizes browser popups for users. It works in three layers:

| Layer | Trigger | Action |
|---|---|---|
| **AWS credential cache** | Credentials expire (12h) but id_token still valid | Re-calls `AssumeRoleWithWebIdentity` silently — no browser |
| **id_token cache** | id_token expires | Opens browser for OneLogin re-authentication |
| **Periodic quota re-check** | Every 30 minutes (configurable) | Re-checks quota using cached id_token — no browser |

The id_token lifetime is controlled in OneLogin under **Security → Token Expiration**. Longer token lifetimes mean fewer browser interruptions. A 1-hour token with keyring storage typically results in one browser popup per day per user.

---

## 15. Troubleshooting

### "Invalid redirect URI"

Ensure the redirect URI in OneLogin is exactly:
```
http://localhost:8400/callback
```
No HTTPS, no trailing slash.

### "Unable to auto-detect provider type"

The domain must be the bare subdomain without `https://`:
```
# Correct
amagi-media-labs.onelogin.com

# Wrong
https://amagi-media-labs.onelogin.com/
```

### "Token is not from a supported provider" (Cognito mode)

The OIDC provider URL registered in the Cognito Identity Pool must exactly match the `iss` claim in the JWT. OneLogin's OIDC v2 issuer is:
```
https://amagi-media-labs.onelogin.com/oidc/2
```
Verify in the CloudFormation stack that the `Url` parameter for the OIDC provider ends with `/oidc/2`.

### "403 InvalidClientTokenId" in Cowork

The `~/.aws/credentials` file has a stale stanza for the `ClaudeCode` profile. Either:
- Switch to keyring storage mode (recommended)
- Or run `credential-process --clear-cache --profile ClaudeCode` to reset it

### Authentication timeout

The browser callback must reach `localhost:8400`. Check:
- No firewall or security tool blocking local port 8400
- The browser opened successfully (check `COGNITO_AUTH_DEBUG=1` logs)
- The OneLogin redirect URI matches exactly

### Quota API returns 401

The id_token has expired. Run `credential-process --clear-cache` to force re-authentication, which will produce a fresh token for quota validation.

### Debug mode

```bash
COGNITO_AUTH_DEBUG=1 credential-process --profile ClaudeCode
```

Prints full token claims, federation steps, and quota check results to stderr.
