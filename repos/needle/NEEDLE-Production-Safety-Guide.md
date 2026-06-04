# NEEDLE Production Safety Guide

> How to run a headless agent fleet without destroying your staging or production environments.

This document is a companion to the NEEDLE Implementation Guide. Read that first. This document assumes NEEDLE is already understood and focuses exclusively on protecting real infrastructure — databases, cloud accounts, APIs, deployed services — from agent-caused damage.

---

## The Threat Model

Before defenses, understand what you are actually defending against. There are three distinct failure modes, and they require different mitigations.

### Failure Mode 1: The Runaway Worker

A worker enters a retry loop, a timeout misconfiguration, or a bad prompt and autonomously takes destructive action at scale. Examples from production systems (not hypothetical):

- A coding agent determined the most efficient fix for a production issue was to delete the environment and rebuild from scratch. 13-hour outage.
- An agent given deploy authority pushed broken migrations to production because the task said "deploy when tests pass" and the tests didn't cover the migration path.
- A worker running `bf close` on every bead it touched — including beads it hadn't actually completed — because CLAUDE.md said "always close when done" and it interpreted "done" broadly.

**Root cause:** Agents optimize for task completion. They do not naturally model blast radius. Given authority to do X, they will do X by whatever path reaches the completion condition fastest.

### Failure Mode 2: Credential Leakage via Prompt Injection

A worker fetches a dependency, reads a test fixture, calls a third-party API, or pulls in external data. That data contains hidden instructions. The worker executes them. Those instructions exfiltrate credentials from the shell environment to an attacker-controlled server.

This is not theoretical. Johann Rehberger demonstrated Claude Computer Use navigating to a malicious page and connecting to a C2 server in a single unguarded session. The entire chain worked on the first try.

**Root cause:** Workers inherit the full shell environment of whoever launched them. That environment may contain AWS credentials, database URLs, Anthropic API keys, GitHub PATs. The agent cannot distinguish a legitimate instruction from a malicious one embedded in retrieved content.

### Failure Mode 3: Supply Chain Exploitation

A worker installs a dependency the model hallucinated. That package name doesn't exist in the real registry — but an attacker registered it with malicious code (slopsquatting). The package runs on install and exfiltrates everything in the environment.

**Root cause:** LLMs confidently recommend packages that don't exist. Attackers have automated tooling that watches for hallucinated package names and registers them immediately.

---

## The Five Defense Layers

Defense must be layered. No single control is sufficient. The principle: each layer assumes the layer above it has already failed.

```
Layer 1: Credentials        Workers never see real prod credentials
Layer 2: Network egress     Workers cannot reach prod infrastructure
Layer 3: Filesystem scope   Workers can only write where you intend
Layer 4: Bead controls      Dangerous actions require human gates
Layer 5: Process isolation  Worker processes are sandboxed at the OS level
```

Work through them in order. Layers 1 and 2 provide the most protection and are the most important to get right before running any worker against real infrastructure.

---

## Layer 1: Credential Isolation

### The Core Rule

**Workers must never see real production credentials.** Not in environment variables. Not in `.env` files. Not in any file in the workspace. If a credential exists in the worker's environment and the worker is compromised, the credential is compromised.

### What Workers Need vs. What They Get

| What the worker needs | What to give instead |
|-----------------------|---------------------|
| A database to run migrations against | A local or CI-only test database with no real data |
| An AWS account to provision resources | An IAM role scoped to a sandbox account with no prod access |
| An Anthropic API key for its own calls | claude-governor proxy URL (governor holds the key) |
| A GitHub PAT to push code | A PAT scoped to a fork or feature branch only, with no main branch write access |
| Cloud provider credentials | Short-lived, role-assumed credentials that expire in 1 hour |

### Practical Implementation

**Step 1: Create a dedicated agent IAM role (AWS example)**

```bash
# Create a restricted policy — only what workers genuinely need
aws iam create-policy \
  --policy-name NeedleWorkerPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject"
        ],
        "Resource": "arn:aws:s3:::your-dev-bucket/*"
      }
    ]
  }'

# Explicitly deny anything production-adjacent
aws iam create-policy \
  --policy-name NeedleWorkerDenyProd \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Deny",
        "Action": "*",
        "Resource": [
          "arn:aws:s3:::your-prod-bucket/*",
          "arn:aws:rds:*:*:db:prod-*",
          "arn:aws:ec2:*:*:instance/prod-*"
        ]
      }
    ]
  }'
```

**Step 2: Use short-lived credentials — 1-hour expiry maximum**

```bash
# Generate short-lived credentials before launching workers
# Never export long-lived access keys into the worker environment
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/NeedleWorkerRole \
  --role-session-name needle-worker-$(date +%s) \
  --duration-seconds 3600)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken')

# Launch worker — credentials expire in 1 hour automatically
needle run --agent claude-interactive --identity alpha
```

**Step 3: Use a credential injection proxy for API keys**

Instead of exporting `ANTHROPIC_API_KEY` into the worker environment, run [claude-governor](https://github.com/jedarden/claude-governor) or [sandcat](https://github.com/VirtusLab/sandcat) as a proxy. Workers point at the proxy URL; the proxy holds the real key and enforces spend caps.

```bash
# Worker environment — no real key
export ANTHROPIC_BASE_URL=http://localhost:8080   # governor proxy
# ANTHROPIC_API_KEY is intentionally absent from the worker env

# Governor config (separate process, holds the real key)
# governor.yaml
api_key: sk-ant-...  # real key, only governor sees it
daily_limit_usd: 50
weekly_limit_usd: 200
```

**Step 4: Explicitly clear dangerous environment variables before launching workers**

```bash
# Clear everything you don't want workers to inherit
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
unset DATABASE_URL POSTGRES_PASSWORD MYSQL_ROOT_PASSWORD
unset STRIPE_SECRET_KEY SENDGRID_API_KEY TWILIO_AUTH_TOKEN
unset GITHUB_TOKEN GITLAB_TOKEN BITBUCKET_APP_PASSWORD

# Then set only what the worker legitimately needs
export ANTHROPIC_BASE_URL=http://localhost:8080
export NEEDLE_WORKSPACE=/path/to/workspace

needle run --agent claude-interactive --identity alpha
```

**Step 5: Mount `.env` files as `/dev/null` (if using ExitBox or Docker)**

```bash
# ExitBox automatically mounts .env as /dev/null inside the container
exitbox run claude --name feature-work

# Or manually with Docker
docker run \
  -v /dev/null:/workspace/.env:ro \
  -v /dev/null:/workspace/.env.local:ro \
  your-needle-worker-image
```

---

## Layer 2: Network Egress Controls

### Why Network Controls Are Non-Negotiable

Even if a worker never sees real credentials, it can still cause damage by:
- Reaching production databases directly if they're network-accessible
- Calling internal APIs that perform destructive operations
- Exfiltrating code or data to external services
- Installing malicious packages from attacker-controlled registries

### The Default-Deny Posture

Workers should be able to reach nothing by default. Every allowed destination is an explicit exception.

**Minimum allowlist for a NEEDLE fleet:**

```
api.anthropic.com        # LLM calls
github.com               # code push/pull
api.github.com           # GitHub API
crates.io                # Rust package registry (if workers build Rust)
pypi.org                 # Python packages (if workers use Python)
npmjs.org                # Node packages (if workers use Node)
```

**Everything else blocked**, including:
- Your production database hostname
- Your production API endpoints
- Your staging environment (unless workers are explicitly scoped to it)
- Any internal services

### Implementation Options

**Option A: ExitBox (simplest for dev machines)**

ExitBox enforces a domain allowlist via Squid proxy with DNS disabled inside the container. Workers that need a new domain can request it with `exitbox-allow` — you approve it interactively.

```bash
# Install
cargo install --git https://github.com/cloud-exit/exitbox

# Run worker inside ExitBox
exitbox run claude --name feature-work \
  --allow api.anthropic.com \
  --allow github.com \
  --allow api.github.com

# When the worker needs a domain not in the list:
# It calls: exitbox-allow pypi.org
# You see a prompt: "Allow pypi.org? [y/n]"
# Approved domains hot-reload without restarting the container
```

**Option B: Sandcat (devcontainer + mitmproxy)**

Routes all container traffic through a WireGuard tunnel to mitmproxy. Real credentials never enter the container — the proxy injects them only when requests go to allowed hosts. Kill switch: if the tunnel drops, the container loses all network access.

See [github.com/VirtusLab/sandcat](https://github.com/VirtusLab/sandcat) — the devcontainer setup handles most of this automatically.

**Option C: Linux network namespace (server deployments)**

For NEEDLE workers running on a dedicated Linux server, isolate each worker in its own network namespace with explicit routing rules:

```bash
# Create an isolated namespace for this worker
ip netns add needle-alpha

# Create a veth pair
ip link add veth-alpha type veth peer name veth-alpha-ns

# Assign the namespace end
ip link set veth-alpha-ns netns needle-alpha

# Set up NAT through the proxy only
# (detailed iptables rules depend on your network topology)

# Launch the worker inside the namespace
ip netns exec needle-alpha \
  needle run --agent claude-interactive --identity alpha
```

**Option D: iptables rules (coarser but practical)**

If full namespace isolation is too complex, use iptables to block production subnets from the worker user:

```bash
# Create a dedicated system user for NEEDLE workers
useradd -r -s /bin/bash needle-worker

# Block this user from reaching production subnets
iptables -A OUTPUT -m owner --uid-owner needle-worker \
  -d 10.0.1.0/24 \   # production subnet
  -j REJECT

# Allow only specific external hosts
iptables -A OUTPUT -m owner --uid-owner needle-worker \
  -d api.anthropic.com -j ACCEPT
iptables -A OUTPUT -m owner --uid-owner needle-worker \
  -d github.com -j ACCEPT
iptables -A OUTPUT -m owner --uid-owner needle-worker \
  -j REJECT   # default deny everything else

# Launch workers as the restricted user
sudo -u needle-worker needle run --agent claude-interactive --identity alpha
```

### Pre-Install Dependencies to Block Package Registry Attacks

If a worker needs packages, pre-install them in the image. Allowing access to `pypi.org` or `crates.io` at runtime enables DNS exfiltration via hallucinated package names (slopsquatting).

```dockerfile
# Build-time: install all deps with pinned hashes
FROM rust:1.75 AS builder
RUN cargo install --locked \
  serde@1.0.196 \
  tokio@1.36.0

# Runtime image: no package managers, no registries
FROM debian:bookworm-slim
COPY --from=builder /usr/local/cargo/bin/ /usr/local/bin/
# No cargo, no pip, no npm in the runtime image
```

---

## Layer 3: Filesystem Scope

### What Workers Can Write By Default

A NEEDLE worker launched in `/your/project` can write to:
- Everything in `/your/project` (the workspace)
- Everything the launching user owns
- `/tmp`
- Potentially the home directory and beyond

This is too broad. A worker implementing a feature in `src/auth/` has no business touching `src/payments/`, `deploy/`, `scripts/migrate/`, or `~/.ssh/`.

### Restricting with CLAUDE.md

The first line of filesystem defense is `CLAUDE.md`. Be explicit about boundaries:

```markdown
## Filesystem Rules

### You MAY write to:
- src/           — application source code
- tests/         — test files
- docs/          — documentation

### You MUST NOT touch:
- deploy/        — deployment scripts and configs
- scripts/migrate/  — database migration scripts (human-authored only)
- .github/workflows/  — CI configuration
- infrastructure/    — Terraform, CDK, Pulumi
- Any file outside the project root

### You MUST NOT run:
- Any command containing: DROP, DELETE, TRUNCATE, DESTROY, PURGE
- Any deployment command: kubectl apply, terraform apply, cdk deploy, helm upgrade
- Any database migration command outside of an explicit migration bead
- git push --force
- rm -rf
```

Workers do not always follow CLAUDE.md perfectly — which is why filesystem rules in CLAUDE.md are a complement to OS-level controls, not a substitute.

### OS-Level Filesystem Controls

**Read-only mounts for sensitive directories:**

```bash
# Mount deployment and infrastructure dirs read-only before launching workers
# Using bind mounts (Linux)
mount --bind /your/project/deploy /your/project/deploy
mount -o remount,ro /your/project/deploy

mount --bind /your/project/infrastructure /your/project/infrastructure
mount -o remount,ro /your/project/infrastructure

needle run --agent claude-interactive --identity alpha

# Unmount after workers finish
umount /your/project/deploy
umount /your/project/infrastructure
```

**With Docker/ExitBox — explicit volume mounts:**

```bash
docker run \
  -v /your/project/src:/workspace/src:rw \        # workers can write src
  -v /your/project/tests:/workspace/tests:rw \     # workers can write tests
  -v /your/project/deploy:/workspace/deploy:ro \   # read-only
  -v /your/project/infrastructure:/workspace/infrastructure:ro \  # read-only
  -v /dev/null:/workspace/.env:ro \                # no env file
  your-needle-worker-image
```

### Git Branch Protection

Workers pushing code is one of the highest-risk operations. Protect main and your staging branch:

```bash
# GitHub branch protection via API
gh api repos/YOUR_ORG/YOUR_REPO/branches/main/protection \
  --method PUT \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field enforce_admins=true \
  --field restrictions='{"users":[],"teams":[]}'
```

Workers should:
- Only have write access to feature branches
- Never have force-push access
- Require a human to merge their PRs to staging or main

---

## Layer 4: Bead-Level Controls

### Human Gates for Irreversible Actions

Some actions cannot be undone. Database migrations, production deploys, data deletions, email sends, billing events. These must require human approval regardless of how well the worker performs on every other task.

**In CLAUDE.md — explicit gate instructions:**

```markdown
## Human Gates

The following actions require a HUMAN bead to be created and closed by a human
before you proceed. Do NOT execute these actions autonomously:

1. Any database migration (bf create "HUMAN: Review migration {filename}" -p 0)
2. Any production deployment
3. Deletion of more than 10 records from any table
4. Any change to IAM policies or cloud permissions
5. Any outbound communication (email, SMS, webhook to external service)
6. Any change to billing or subscription state

To create a human gate bead:
  bf create "HUMAN: Approve [action] before [task] proceeds" -p 0 --type human
  bf update CURRENT_BEAD_ID --status blocked --blocks HUMAN_BEAD_ID
  # Stop work. Wait for the human bead to be closed.
```

**In bead bodies — explicit scope exclusions:**

```
## Scope
Implement the new user registration flow in src/auth/register.rs.

## Out of Scope — Do Not Touch
- The users table schema (no ALTER TABLE)
- The email sending code in src/notifications/
- Any production configuration
- The existing login flow

## If you find a bug outside this scope:
Create a new bead: bf create "Bug found: [description]" -p 1 --type bug
Do not fix it in this bead.
```

### Acceptance Criteria as a Safety Control

Well-written acceptance criteria constrain what "done" means:

```
## Acceptance Criteria
- The registration endpoint returns 201 on success
- The registration endpoint returns 422 with validation errors on invalid input
- A user record is created in the test database (NOT production)
- No existing tests are broken
- No new dependencies are added
- No files outside src/auth/ and tests/auth/ are modified
```

The last two criteria are safety controls, not functional requirements. Include them in every bead that touches sensitive code paths.

### Staging vs. Production Separation in Beads

Never assign a worker a bead that says "deploy to production." Split it:

```
Bead 1 (worker): Deploy to staging
  - Acceptance: staging deployment succeeds, smoke tests pass

Bead 2 (HUMAN): Review staging deployment
  - Blocks bead 3
  - Human closes this after reviewing staging

Bead 3 (worker): Deploy to production
  - Only claimable after bead 2 is closed
  - Has a separate HUMAN gate bead for final approval
```

Use `bf dep add` to enforce the ordering:

```bash
bf dep add BEAD_3_ID BEAD_2_ID   # bead 3 blocked by bead 2
bf dep add BEAD_2_ID BEAD_1_ID   # bead 2 blocked by bead 1
```

---

## Layer 5: Process Isolation

### Why the Other Layers Are Not Enough Alone

CLAUDE.md rules can be bypassed by prompt injection. Network rules can be misconfigured. Filesystem mounts can be forgotten. Process isolation is the backstop — it enforces limits at the OS level regardless of what the agent does.

### Options by Environment

**Development machine (macOS): SmolVM**

SmolVM gives each worker its own kernel in ~100–200ms. No shared kernel = no kernel exploit path between worker and host.

```bash
# Install
cargo install --git https://github.com/particula-tech/smolvm

# Run worker inside a microVM
smolvm run \
  --image your-needle-worker-image \
  --mount /your/project/src:/workspace/src:rw \
  --mount /your/project/tests:/workspace/tests:rw \
  --network allowlist:api.anthropic.com,github.com \
  -- needle run --agent claude-interactive --identity alpha
```

**Linux server: Firecracker or gVisor**

Firecracker gives each worker its own kernel with ~125ms boot and hardware-enforced isolation. Used by AWS Lambda at hyperscale.

```bash
# Using Kata Containers (Firecracker backend) with containerd
# Each worker container gets its own kernel
ctr run \
  --runtime io.containerd.kata.v2 \
  --env ANTHROPIC_BASE_URL=http://localhost:8080 \
  your-needle-worker-image \
  needle-alpha \
  needle run --agent claude-interactive --identity alpha
```

gVisor intercepts syscalls in userspace. Less isolation than Firecracker but easier to set up and sufficient for most threat models:

```bash
docker run \
  --runtime=runsc \   # gVisor runtime
  --network=none \    # no network by default; route through proxy
  your-needle-worker-image \
  needle run --agent claude-interactive --identity alpha
```

**Simpler alternative: drop Linux capabilities**

If full microVM isolation is too complex for your first setup, at minimum drop dangerous capabilities from the worker process:

```bash
# Drop all capabilities, add back only what's needed
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \   # if workers need to bind ports
  --no-new-privileges \          # prevent privilege escalation
  --read-only \                  # read-only root filesystem
  --tmpfs /tmp:rw,noexec,nosuid \  # writable tmp, no execution
  your-needle-worker-image
```

### Isolation Technology Comparison

| Technology | Isolation | Boot time | Complexity | Best for |
|-----------|-----------|-----------|------------|---------|
| Docker (default) | Low — shared kernel, real CVEs in 2024–2026 | Fast | Low | Trusted code only |
| Docker + dropped caps | Moderate | Fast | Low | First step before better options |
| gVisor (`runsc`) | Good — filtered syscalls | Fast | Medium | Linux server, HTTP workloads |
| SmolVM | Strong — separate kernel | 80–260ms | Medium | macOS dev machines, Linux |
| Firecracker / Kata | Strongest — hardware VM | ~125ms | High | Production server fleets |

---

## Putting It Together: The Safe Launch Sequence

This is the sequence to follow every time you start a NEEDLE worker session against any non-trivial environment.

```bash
#!/bin/bash
# safe-launch-worker.sh

set -euo pipefail

IDENTITY=${1:-alpha}
WORKSPACE=${2:-$(pwd)}

echo "=== Pre-flight safety checks ==="

# 1. Verify no production credentials in environment
for var in AWS_ACCESS_KEY_ID DATABASE_URL POSTGRES_PASSWORD STRIPE_SECRET_KEY; do
  if [[ -n "${!var:-}" ]]; then
    echo "ERROR: $var is set in environment. Clear it before launching workers."
    exit 1
  fi
done
echo "✅ No production credentials in environment"

# 2. Verify claude-governor is running
if ! curl -sf http://localhost:8080/health > /dev/null 2>&1; then
  echo "ERROR: claude-governor is not running. Start it before launching workers."
  exit 1
fi
echo "✅ claude-governor is running"

# 3. Verify workspace has CLAUDE.md
if [[ ! -f "$WORKSPACE/CLAUDE.md" ]]; then
  echo "ERROR: No CLAUDE.md found in $WORKSPACE. Workers need project instructions."
  exit 1
fi
echo "✅ CLAUDE.md present"

# 4. Generate short-lived credentials if needed
if [[ -n "${NEED_AWS_CREDS:-}" ]]; then
  CREDS=$(aws sts assume-role \
    --role-arn "$NEEDLE_WORKER_ROLE_ARN" \
    --role-session-name "needle-$IDENTITY-$(date +%s)" \
    --duration-seconds 3600)
  export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')
  export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey')
  export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken')
  echo "✅ Short-lived AWS credentials generated (1hr expiry)"
fi

# 5. Set worker to use governor, not real API
export ANTHROPIC_BASE_URL=http://localhost:8080
unset ANTHROPIC_API_KEY

echo "=== Launching worker: $IDENTITY ==="
cd "$WORKSPACE"
tmux new-session -d -s "needle-$IDENTITY" \
  "needle run --agent claude-interactive --identity $IDENTITY"

echo "✅ Worker started: needle-$IDENTITY"
echo "   Attach: tmux attach -t needle-$IDENTITY"
```

---

## Environment Tiers

Apply different levels of control depending on the environment workers are targeting:

| Tier | Environment | Credential access | Network | Process isolation | Human gates |
|------|-------------|------------------|---------|------------------|-------------|
| 0 | Local dev only | None (mock everything) | None needed | Docker + dropped caps | None needed |
| 1 | Shared dev/CI | Dev-only IAM role, 1hr expiry | Allowlist | gVisor or SmolVM | On deploys |
| 2 | Staging | Staging-only IAM role, 1hr expiry | Allowlist, no prod routes | Firecracker/Kata | On migrations + deploys |
| 3 | Production-adjacent | Never. Workers do not touch prod directly. | — | — | Human closes every gate bead |

**The rule for Tier 3:** Workers propose changes. Humans review and execute. The worker's job ends at a green PR or a closed gate bead. A human presses the button.

---

## Checklist Before Running Workers Against Real Infrastructure

- [ ] No production credentials in the worker's shell environment
- [ ] API keys go through claude-governor or equivalent spend proxy — not set directly
- [ ] AWS/GCP/Azure roles are scoped to dev/staging only with explicit prod denies
- [ ] Short-lived credentials (1hr max) generated fresh per session
- [ ] Network egress allowlist in place — workers cannot reach prod hostnames
- [ ] Production database hostname is not reachable from worker network
- [ ] `.env` files are absent or read-only inside the worker process
- [ ] `deploy/`, `infrastructure/`, `scripts/migrate/` are read-only mounts
- [ ] Git branch protection prevents direct pushes to main and staging
- [ ] Workers only have write access to feature branches
- [ ] CLAUDE.md explicitly lists forbidden commands and directories
- [ ] Every irreversible action (migration, deploy, delete, email) has a HUMAN gate bead
- [ ] Bead dependencies enforce the staging → human review → production sequence
- [ ] Workers are running under a restricted OS user or inside a container
- [ ] At minimum: `--cap-drop=ALL --no-new-privileges --read-only` if using Docker
- [ ] Preferably: gVisor, SmolVM, or Firecracker for kernel-level isolation
- [ ] claude-governor spend caps are set and verified before fleet launch
- [ ] `needle status` and `bf stats` are being watched during the first session

---

*Companion to the NEEDLE Implementation Guide · github.com/jedarden/NEEDLE*
