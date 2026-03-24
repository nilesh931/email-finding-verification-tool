# Build Your Own Email Finding/Verification System for $5/Month

A self-hosted email verification pipeline that finds and validates business emails at **zero per-email cost**. Runs on three free tools and a $5/month VPS. No expensive third-party API subscriptions, no per-result billing, no rate limits imposed by someone else's infrastructure.

## What This Does

Given a list of contacts with `first_name`, `last_name`, and `company_domain`, this system:

- **Finds** the correct email pattern (tries 8 common formats)
- **Verifies** each email against the actual mail server via SMTP
- **Bypasses** corporate email gateways (Proofpoint, Mimecast, Barracuda) using Microsoft 365 identity checks
- **Scores** every result with a confidence level: `high`, `medium`, or `low`
- **Handles** catch-all domains, dead domains, and greylisting automatically

## Performance

| Metric | Value |
|--------|-------|
| Speed | ~1,500 contacts/min (50 workers) |
| 10,000 contacts | ~23 minutes end to end |
| Cost per email | $0.00 |
| Monthly infrastructure | ~$5 |
| Deliverable email rate | ~31% (high 10%, medium 8%, low 13%) |

Benchmarked on a Hetzner CX23 (2 vCPU, 4GB RAM) processing 16,148 contacts across 4,503 domains.

## The Stack

| Component | Purpose | Cost |
|-----------|---------|------|
| **SMTP Verifier** | Custom Python service on your VPS. Talks directly to mail servers. | Free (self-hosted) |
| **Reacher** | Open-source Docker container for edge cases. | Free (open-source) |
| **M365 Identity Check** | Microsoft's public API. Bypasses corporate gateways. | Free (no auth needed) |
| **Hetzner CX23 VPS** | Runs everything. Helsinki or Nuremberg. | $4.91/month |
| **Cheap Domain** | SMTP sender identity (DNS, SPF, rDNS). | ~$2/year |

## Quick Start

### Prerequisites

- A [Hetzner](https://hetzner.com) CX23 VPS ($4.91/month)
- A cheap domain from Porkbun or Namecheap (~$2/year)
- [Cloudflare](https://cloudflare.com) DNS (free plan)
- [Claude Code](https://claude.ai) (writes the pipeline code for you)

### 1. Set Up the VPS

```bash
# SSH into your new server
ssh root@YOUR_VPS_IP

# Install dependencies
apt update && apt install -y docker.io python3-dnspython

# Deploy Reacher
docker run -d --name reacher --restart always \
  -e RCH__FROM_EMAIL=verify@yourdomain.com \
  -e RCH__HELLO_NAME=mail1.yourdomain.com \
  -e RCH__HTTP_HOST=0.0.0.0 \
  -p 8080:8080 reacherhq/backend:latest
```

> **Critical:** Submit a Hetzner support ticket to unblock port 25. Without this, SMTP verification won't work. Approval typically arrives within 24 hours.

### 2. Configure DNS

Add your domain to Cloudflare. Three records are required:

| Record | Value |
|--------|-------|
| A Record | `mail1.yourdomain.com` -> VPS IP |
| TXT (SPF) | `v=spf1 ip4:YOUR_VPS_IP ~all` |
| Reverse DNS | VPS IP -> `mail1.yourdomain.com` (set in Hetzner console) |

All three are non-negotiable. Missing any one causes mail servers to reject your probes.

### 3. Install the Claude Code Skill

```bash
# Copy the skill file to your Claude Code commands directory
cp email-verify.md ~/.claude/commands/email-verify.md
```

Now invoke `/email-verify` in any Claude Code session and it loads the full system context automatically.

### 4. Run Your First Pipeline

Tell Claude Code:

> "Build an async Python email verification pipeline using the /email-verify skill. My CSV has first_name, last_name, corporate_website, linkedin_url. Find and verify emails for contacts missing email. Write results back to CSV. Use Step Zero first, then the waterfall. Make it resumable."

Claude Code reads your skill and builds the complete script.

## How It Works

### The 5-Step Pipeline

```
Step 0: Pre-filter
    Filter disposable domains, empty names, invalid domains.
    Zero API calls. Instant.
         |
Step 1: Domain Classification (25 workers on VPS, ~4 min per 3,000 domains)
    One fake SMTP probe per domain.
    Classifies as: responsive / catch_all / no_mx / m365 / gateway
         |
Step 2: Route Contacts
    responsive + catch_all  -->  SMTP Stream
    m365 + gateway          -->  M365 Stream
    no_mx                   -->  Skip
         |
Step 3: Find Emails (two parallel streams, 50 workers each)
    |                                    |
    SMTP Stream                          M365 Stream
    8 patterns + 1 fake probe            4 patterns via Microsoft API
    per SMTP session                     GetCredentialType endpoint
    |                                    |
Step 4: Merge + Apply
    Combine results with confidence scores.
    Write to CSV.
```

### Confidence Scoring

Every found email gets a confidence level:

| Confidence | Meaning | Reliability |
|-----------|---------|-------------|
| **high** | SMTP 250 on responsive server + fake probe rejected | 95%+ |
| **medium** | M365 Azure AD identity confirmed | 85% |
| **low** | Catch-all domain, first.last@ assigned | 50% pattern accuracy, 100% deliverable |

### The 8 Email Patterns (tried in order)

| # | Pattern | Frequency |
|---|---------|-----------|
| 1 | first.last@domain | ~50% |
| 2 | flast@domain | ~20% |
| 3 | first@domain | ~8% |
| 4 | firstlast@domain | ~6% |
| 5 | first_last@domain | ~5% |
| 6 | last.first@domain | ~4% |
| 7 | firstl@domain | ~3% |
| 8 | f.last@domain | ~2% |

### The M365 Secret Weapon

Corporate email gateways block SMTP probes. The M365 Identity Check sidesteps all of them using Microsoft's free, unauthenticated public API:

```
Step 1: GET https://login.microsoftonline.com/getuserrealm.srf?login=anyone@domain.com
        If NameSpaceType == "Managed" --> domain uses M365

Step 2: POST https://login.microsoftonline.com/common/GetCredentialType
        Body: {"Username": "first.last@domain.com"}
        IfExistsResult == 0 --> user exists, email is valid
```

This works because Microsoft's auth system must resolve user identities before presenting the login screen. That lookup happens before any gateway policy applies.

## Key Optimizations

These aren't theoretical. Every one was measured against the alternative.

| Optimization | Before | After | Speedup |
|-------------|--------|-------|---------|
| VPS-side execution (skip HTTP API) | 12/min | 1,500/min | 125x |
| Batched SMTP (one connection, 8 RCPT TO) | 1 connection per pattern | 1 connection per contact | 5-6x |
| Two parallel streams (SMTP + M365) | Sequential | Parallel | 2x |
| Inline fake-probe (classify + verify in one session) | Separate Step Zero pass | Single SMTP session | Saves entire pass |
| Chunked futures (500 at a time) | 16k futures at once (hangs) | Incremental progress | Actually works |

## Failure Handling

Each step has its own failure mode and recovery. No step blocks the others indefinitely.

- **VPS unreachable:** Check Hetzner console. Reboot or redeploy.
- **SMTP stream stalls:** Kill and re-run. Checkpoint resumes from last saved chunk.
- **M365 rate limited (429):** Reduce workers to 20, add 1s delay.
- **One stream stuck, other done:** Save finished results. Kill stuck stream. Re-run only the stuck one.
- **Circuit breaker:** If a domain fails 3 times, blacklist it for the rest of the run.

## Non-Negotiable Rules

1. **Never run SMTP from your local machine.** Port 25 is blocked on residential ISPs. 30-40% false failure rate.
2. **Always include a fake probe.** A 250 on a catch-all server means nothing without confirming the server rejects fakes.
3. **Always assign confidence scores.** No unscored results. Ever.
4. **Always run bulk operations on the VPS.** Never route 100+ checks through the HTTP API.
5. **Always split SMTP and M365 into parallel streams.** Different speed profiles cause mutual blocking.
6. **Always process in chunks of 500.** Never submit all futures at once.
7. **Cache everything.** MX records, M365 realm checks, domain classifications. Never re-probe what you already know.

## File Structure

```
.
├── README.md                  # This file
├── email-verify.md            # Claude Code skill file (copy to ~/.claude/commands/)
├── LICENSE                    # MIT
└── docs/
    └── architecture.pdf       # Visual guide (the Gamma deck)
```

## Installation

```bash
# Clone the repo
git clone https://github.com/nilesh931/email-finding-verification-tool.git

# Install the skill
cp email-finding-verification-tool/email-verify.md ~/.claude/commands/

# Verify it works
# Open Claude Code and type: /email-verify
```

## License

MIT. Use it however you want.

## Author

Built by [Nilesh Patil](https://www.linkedin.com/in/go-to-market-engineer/). Questions? nilesh@prospectspulse.com
