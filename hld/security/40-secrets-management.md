# 40. Secrets Management

## What is Secrets Management?

Secrets are sensitive configuration values: database passwords, API keys, TLS private keys, encryption keys, OAuth client secrets. Secrets management is the practice of **securely storing, distributing, rotating, and auditing access to secrets**.

**The problem:** Secrets must get from where they're stored to where they're needed (running application) without being exposed in:
- Source code (git history)
- Environment variables (process listings, crash dumps)
- Container images
- CI/CD logs
- Config files on disk

---

## Q1: What are the most common secrets mistakes?

### 1. Hardcoded secrets in source code
```python
# ❌ NEVER DO THIS
db_password = "super_secret_password_123"
api_key = "sk-prod-abc123def456"
```

Git history is permanent. Even after removing the secret, it exists in every clone, fork, and historical commit. Tools like `truffleHog`, `git-secrets`, and GitHub's secret scanning detect this automatically.

### 2. Secrets in environment variables
```bash
# ❌ Risky — visible in process list, crash dumps, CI/CD logs
export DB_PASSWORD="super_secret"
```

`/proc/<pid>/environ` on Linux exposes all environment variables to any process running as the same user. Crash reports and log aggregators often capture env vars.

### 3. Secrets in Docker images
```dockerfile
# ❌ Secret baked into image layer — visible with docker history
RUN curl -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/
```

Every `RUN` layer is preserved in the image. Anyone with the image can extract the secret.

### 4. Secrets in config files on disk
```yaml
# ❌ config.yaml committed to repo or readable by wrong processes
database:
  password: "super_secret"
```

---

## Q2: What is HashiCorp Vault?

**Vault** is the most widely used open-source secrets management tool. It stores secrets encrypted, controls access with policies, and provides audit logging.

### Core Concepts

**Secrets Engines:** Vault supports multiple backends for different secret types:
- `kv` (Key-Value) — store static secrets
- `database` — dynamically generate DB credentials
- `pki` — generate TLS certificates
- `aws` — generate temporary AWS credentials
- `transit` — encryption as a service (encrypt/decrypt without storing data)

**Auth Methods:** How clients authenticate to Vault:
- `kubernetes` — pods authenticate using their service account JWT
- `aws` — EC2 instances authenticate using instance metadata
- `approle` — role ID + secret ID (for CI/CD pipelines)
- `ldap`, `github`, `oidc` — for human users

**Policies:** ACL rules controlling what a client can access:
```hcl
# payment-service policy
path "secret/data/payment/*" {
  capabilities = ["read"]
}

path "database/creds/payment-db-role" {
  capabilities = ["read"]
}

# Cannot read other services' secrets
path "secret/data/auth-service/*" {
  capabilities = ["deny"]
}
```

### Dynamic Secrets (killer feature)

Instead of storing a static DB password, Vault **generates a short-lived credential on demand**:

```
Payment Service → Vault: "I need DB credentials"
Vault → checks policy: payment-service is allowed
Vault → creates temporary MySQL user: username=v-payment-abc123, password=xyz789
Vault → returns credentials with TTL=1 hour
Payment Service → connects to MySQL with temporary credentials

After 1 hour: Vault automatically revokes the temporary user
```

**Advantages:**
- No long-lived shared passwords
- Each service gets unique credentials — compromise of one doesn't affect others
- Vault can auto-revoke if the service is compromised
- Full audit trail: "at 14:32, payment-service requested DB credentials"

### Vault Architecture

```
Client → [Vault Agent (sidecar)] → [Vault Server cluster] → [Storage Backend (Consul/DynamoDB/etcd)]
                                           │
                                    [Unseal keys / Auto-unseal (AWS KMS)]
```

**Vault seal/unseal:** Vault starts sealed (encrypted). Requires unseal keys (or cloud KMS) to decrypt the master key and become operational. This prevents cold-boot attacks on the storage backend.

---

## Q3: How does AWS Secrets Manager work?

**AWS Secrets Manager** is a managed service (no infrastructure to run). Store secrets, rotate them automatically, access them via API.

```python
import boto3

client = boto3.client('secretsmanager', region_name='us-east-1')

# Retrieve secret
response = client.get_secret_value(SecretId='prod/payment-service/db')
secret = json.loads(response['SecretString'])
# secret = {"username": "payment_user", "password": "rotated_password_xyz"}
```

### Automatic Rotation

Configure a Lambda function to rotate secrets on a schedule:

```
Every 30 days:
1. Secrets Manager calls rotation Lambda
2. Lambda creates new password in RDS
3. Lambda updates the secret in Secrets Manager
4. Lambda tests the new credentials
5. Lambda marks rotation complete
6. Old password remains valid during transition window
7. Old password deleted after transition
```

**Supported automatic rotation:** RDS (MySQL, PostgreSQL, Oracle), Redshift, DocumentDB, others via custom Lambda.

### AWS Secrets Manager vs SSM Parameter Store

| | Secrets Manager | SSM Parameter Store |
|---|---|---|
| Cost | $0.40/secret/month | Free (Standard), $0.05/advanced |
| Automatic rotation | Yes (built-in) | No (manual Lambda) |
| Cross-account access | Easy | Requires more setup |
| Secret size | 64KB | 4KB (standard), 8KB (advanced) |
| Use case | Sensitive secrets, DB passwords | Config values, some secrets |

**Rule of thumb:** Use Secrets Manager for actual secrets (passwords, keys). Use Parameter Store for non-sensitive config that should be centrally managed.

---

## Q4: Secret injection patterns — how do secrets reach the application?

### Pattern 1: Direct API Call (application fetches at startup)

```python
# Application fetches secret on startup
import boto3

def get_db_config():
    client = boto3.client('secretsmanager')
    secret = json.loads(
        client.get_secret_value(SecretId='prod/db')['SecretString']
    )
    return secret

db_config = get_db_config()  # called once at startup
```

**Pros:** Simple, works everywhere
**Cons:** Secret lives in memory, must handle rotation (reconnect when password changes)

---

### Pattern 2: Vault Agent Sidecar (file injection)

Vault Agent runs as a sidecar, writes secrets to an in-memory filesystem (`tmpfs`):

```yaml
# Kubernetes pod with Vault Agent sidecar
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-inject-secret-db: "secret/data/payment/db"
  vault.hashicorp.com/agent-inject-template-db: |
    {{- with secret "secret/data/payment/db" -}}
    DB_HOST={{ .Data.data.host }}
    DB_PASSWORD={{ .Data.data.password }}
    {{- end }}
```

Vault Agent writes `/vault/secrets/db` file. Application reads from file. Vault Agent automatically refreshes when secret rotates.

**Pros:** Application doesn't need Vault SDK, secrets never in env vars, auto-refresh on rotation
**Cons:** Requires Vault Agent sidecar, file still exists in pod (though tmpfs = memory only)

---

### Pattern 3: Kubernetes Secrets + External Secrets Operator

**External Secrets Operator (ESO)** syncs secrets from AWS Secrets Manager/Vault/GCP Secret Manager into Kubernetes Secrets:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: payment-db-k8s-secret  # creates this k8s Secret
  data:
  - secretKey: db-password
    remoteRef:
      key: prod/payment/db
      property: password
```

ESO pulls from AWS Secrets Manager every hour and updates the Kubernetes Secret. Pod mounts the k8s Secret.

---

## Q5: Secret rotation best practices

### 1. Automate rotation — never manual
Manual rotation = someone forgets = old secret valid for months/years.

### 2. Short TTLs for dynamic secrets
Vault dynamic DB credentials: 1 hour TTL. If stolen, useless after 1 hour.

### 3. Rotation without downtime
Blue-green rotation:
1. Create new secret (new DB password)
2. Update application to accept both old + new
3. Rotate: update all app instances to use new secret
4. Revoke old secret

### 4. Separate secrets per environment
`dev/db/password`, `staging/db/password`, `prod/db/password` — production secrets must never be accessible to developers directly.

### 5. Audit everything
Every secret access should be logged: who, what, when, from where.

---

## Q6: Secret scanning in CI/CD

Prevent secrets from entering the codebase:

**Pre-commit hooks:**
```bash
# .pre-commit-config.yaml
repos:
- repo: https://github.com/trufflesecurity/trufflehog
  hooks:
  - id: trufflehog
```

**GitHub Secret Scanning:** GitHub automatically scans all pushes for known secret patterns (AWS keys, GitHub tokens, Stripe keys, etc.) and alerts/blocks if found.

**GitLeaks in CI pipeline:**
```yaml
- name: Scan for secrets
  run: gitleaks detect --source . --exit-code 1
```

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| AWS Secrets Manager cost | $0.40/secret/month + $0.05/10k API calls |
| Vault dynamic secret TTL (typical) | 1 hour |
| Recommended rotation frequency (static) | 90 days max |
| Let's Encrypt cert auto-renewal | Every 60 days (90-day validity) |
| Secret size limit (AWS Secrets Manager) | 64 KB |

---

## Interview Q&A

**Q: A developer accidentally committed an API key to a public GitHub repo. What do you do?**
A: Assume it's compromised immediately — treat exposure as certain, not possible. Step 1: Revoke/rotate the key NOW (seconds matter — bots scan GitHub in real time). Step 2: Audit the key's usage logs — has it already been misused? Step 3: Remove from git history (`git filter-branch` or `git filter-repo`) and force-push. Step 4: Add the secret pattern to git-secrets/pre-commit hooks to prevent recurrence. Step 5: Post-mortem: why was it hardcoded? Implement secrets management to prevent it.

**Q: How do you handle secret rotation without application downtime?**
A: Two approaches. (1) Support dual credentials during rotation: before rotating, configure the app to try the new credential and fall back to old — then rotate — then remove old. (2) For apps using Vault Agent or ESO, rotation is transparent: the agent refreshes the file, and the app periodically re-reads credentials (e.g., on next connection pool validation). Database connection pools handle this gracefully — when a connection fails (old password), it reconnects with the refreshed credential. Design apps to reload secrets on a signal (SIGHUP) or periodically rather than caching at startup forever.

**Q: Why is Vault's dynamic secret generation better than storing a shared DB password?**
A: A shared static password is a single point of compromise — if any service that knows it is breached, all are at risk. Dynamic secrets generate unique, short-lived credentials per request per service. If payment-service is breached, its credential is unique to it and expires in 1 hour. Other services are unaffected. The "blast radius" of a credential compromise is bounded. Also: dynamic secrets provide a perfect audit trail — you know exactly which service accessed the DB at what time, because each credential is unique to a request.