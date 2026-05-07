# OneLogin Setup Guide for Amazon Bedrock Integration

This guide walks you through configuring OneLogin as the OIDC identity provider for Claude Code with Amazon Bedrock.

## Prerequisites

- OneLogin account with administrator access
- AWS account with IAM and CloudFormation permissions
- Amazon Bedrock activated in your target regions

---

## 1. Create an OIDC Application in OneLogin

1. Log in to your OneLogin Admin Console at `https://amagi-media-labs.onelogin.com/admin`
2. Navigate to **Applications** → **Applications**
3. Click **Add App**
4. Search for **OpenID Connect (OIDC)** and select it
5. Give it a name: `Amazon Bedrock CLI Access`
6. Click **Save**

---

## 2. Configure the Application

### Configuration Tab

Under the **Configuration** tab:

- **Redirect URI**: `http://localhost:8400/callback`
- **Post Logout Redirect URI**: `http://localhost:8400/logout` (optional)
- **Login URL**: leave blank

### SSO Tab

Under the **SSO** tab:

- **Application Type**: `Web`
- **Authentication Method**: `none` — this enables PKCE-only (public client) mode
- Note your **Client ID** — this is the only credential you need

> **No client secret required.** Setting Authentication Method to `none` enables the OAuth 2.0 Authorization Code + PKCE flow. The credential provider sends only the `client_id` and PKCE `code_verifier` during token exchange — no secret is ever stored or distributed to end users.

---

## 3. Add Users / Groups

### Assign Users

1. Go to the **Users** tab on your application
2. Click **New User** or use **Add Group** to assign existing directory groups
3. Ensure the users who need Bedrock access are assigned

### (Optional) Add Groups Claim for Quota Policies

If you want group-based quota monitoring:

1. Go to **Parameters** tab on the application
2. Click **+** to add a parameter
3. Set **Field name**: `groups`
4. Set **Value**: `User Roles` (or a specific OneLogin role attribute)
5. Check **Include in SAML assertion** and **Multi-value parameter**
6. Click **Save**

---

## 4. Collect Required Information

| Parameter | Your Value | Example |
|---|---|---|
| **OneLogin Domain** | Your subdomain | `amagi-media-labs.onelogin.com` |
| **Client ID** | From SSO tab | `abc123def456...` |

> No client secret needed — the app uses PKCE (public client mode).

---

## 5. Deploy with ccwb

Run the setup wizard:

```bash
poetry run ccwb init
```

When prompted:

```
Enter your OIDC provider domain: amagi-media-labs.onelogin.com
# Auto-detected as: OneLogin

Enter your Client ID: <your-client-id>
```

The tool auto-detects the provider type from the `.onelogin.com` domain and selects `bedrock-auth-onelogin.yaml` automatically.

Then deploy:

```bash
poetry run ccwb deploy auth
poetry run ccwb package
```

---

## 6. OIDC Endpoints Used

The credential provider uses OneLogin's OIDC v2 endpoints:

| Endpoint | URL |
|---|---|
| Authorization | `https://amagi-media-labs.onelogin.com/oidc/2/auth` |
| Token | `https://amagi-media-labs.onelogin.com/oidc/2/token` |
| Scopes | `openid profile email` |

---

## 7. Verify the Setup

After packaging and distributing to a test user:

```bash
./credential-process --profile ClaudeCode
```

A browser window will open to `amagi-media-labs.onelogin.com` for login. On success, AWS credentials are printed to stdout and cached for the session.

---

## Troubleshooting

### "Invalid redirect URI" error

Ensure the redirect URI in OneLogin is exactly:
```
http://localhost:8400/callback
```
No trailing slash. No HTTPS.

### "Unknown provider type" error

Make sure `provider_domain` in your config is set to the bare domain without `https://`:
```
amagi-media-labs.onelogin.com
```
Not `https://amagi-media-labs.onelogin.com/`.

### Token not accepted by Cognito Identity Pool

Verify the **OIDC Provider URL** in the CloudFormation stack matches the issuer in the JWT exactly. OneLogin's issuer for OIDC v2 is:
```
https://<subdomain>.onelogin.com/oidc/2
```

---

## Next Steps

1. Run `poetry run ccwb test --api` to verify Bedrock connectivity
2. See [Quota Monitoring](../QUOTA_MONITORING.md) for per-user token limits
3. See [Cost Attribution](../COST_ATTRIBUTION.md) for per-user AWS cost tracking
