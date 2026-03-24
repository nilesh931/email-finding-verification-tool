# Email Verification — In-House Tools Only

Technical guidance for verifying email addresses using our self-hosted VPS infrastructure + Microsoft 365 identity checks. No third-party paid APIs.

## Section 1: Health Check (run FIRST, every time)

Before doing anything, verify all 3 services are up. If any is down, fix it before proceeding.

### 1. Reacher API

```bash
curl -s http://YOUR_VPS_IP:8080/v0/check_email -X POST -H "Content-Type: application/json" -d '{"to_email":"test@google.com"}'
```

Should return JSON with `is_reachable`. The root `/` endpoint may return empty, always test with the actual check_email endpoint. If down:

```bash
ssh root@YOUR_VPS_IP   # password: YOUR_PASSWORD
docker restart reacher
```

### 2. SMTP Verifier API

```bash
curl -s http://YOUR_VPS_IP:8081/
```

Should return `{"service": "smtp-verifier", "status": "ok"}`. If down:

```bash
ssh root@YOUR_VPS_IP
systemctl restart smtp-verifier
```

### 3. M365 API

```bash
curl -s "https://login.microsoftonline.com/getuserrealm.srf?login=test@microsoft.com"
```

Should return JSON with `NameSpaceType`. This is Microsoft's public endpoint, no auth needed.

### If VPS is unreachable

SSH times out entirely. Follow Section 5 to deploy a new server.

---

## Section 2: Available Tools

### Tool 1: Reacher API (VPS, port 8080)

- **Endpoint**: `POST http://YOUR_VPS_IP:8080/v0/check_email`
- **Body**: `{"to_email": "email@domain.com"}`
- **Response fields**: `is_reachable` (safe/risky/invalid/unknown), `smtp.is_deliverable`, `smtp.is_catch_all`, `smtp.can_connect_smtp`
- **Runtime**: Docker container, auto-restarts on crash
- **Strengths**: Mature tool, handles edge cases well
- **Weaknesses**: Sometimes returns `unknown` where SMTP Verifier succeeds
- **Use for**: Fallback when SMTP Verifier fails on specific domains

### Tool 2: SMTP Verifier API (VPS, port 8081)

- **Endpoint**: `POST http://YOUR_VPS_IP:8081`
- **Body**: `{"email": "email@domain.com", "from_email": "verify@yourdomain.com", "helo": "yourdomain.com"}`
- **Response fields**: `status` (valid/invalid/no_mx/error), `deliverable` (true/false), `smtp_code` (250/550/etc.), `mx` (mail server hostname)
- **Runtime**: Systemd service (`smtp-verifier`), survives reboots
- **Strengths**: Catches catch-all domains better, simpler response format
- **Weaknesses**: No built-in rate limiting
- **Use for**: Small batches (< 100 contacts) run from client, domain classification

### Tool 3: Direct SMTP via Python smtplib (VPS-side execution)

For batches over 100 contacts, skip the HTTP API entirely. Upload a Python script to the VPS and run SMTP checks directly using `smtplib`. This eliminates the HTTP round trip and is 5-10x faster.

```python
import smtplib
with smtplib.SMTP(mx_host, 25, timeout=10) as smtp:
    smtp.ehlo("yourdomain.com")
    smtp.mail("verify@yourdomain.com")
    code, _ = smtp.rcpt(email)  # 250 = valid, 550 = rejected
```

- **Requires**: `python3-dnspython` on VPS for MX lookups
- **Workers**: 50 concurrent is safe on a CX23. Port 25 connections are lightweight.
- **Use for**: All bulk operations (100+ contacts)

### Tool 4: M365 Identity Check (no server needed)

Two-step process, zero cost, no auth:

**Step 1** — Check if domain is Microsoft 365 Managed:

```
GET https://login.microsoftonline.com/getuserrealm.srf?login={email}
```

If `NameSpaceType == "Managed"`, proceed to Step 2.

**Step 2** — Check if user exists:

```
POST https://login.microsoftonline.com/common/GetCredentialType
Body: {"Username": "{email}"}
```

- `IfExistsResult == 0` -> user exists, email is valid
- `IfExistsResult == 1` -> not found
- Bypasses all email gateways (Proofpoint, Mimecast, Barracuda)
- Only works for Microsoft 365 / Azure AD managed domains
- **CRITICAL**: M365 confirms the user EXISTS in Azure AD, but does not confirm SMTP deliverability. Always cross-verify with SMTP when possible. Never use M365 alone as the only verification.

---

## Section 3: Execution Modes

### Mode A: Small Batch (< 100 contacts)

Use the HTTP APIs (Tools 1 and 2) from the client. Step Zero via API, then pattern waterfall via API. Acceptable speed.

### Mode B: Bulk Batch (100+ contacts) — ALWAYS USE THIS

Upload a Python script to the VPS and run directly. Two parallel streams.

**Architecture:**
```
VPS runs two processes simultaneously:
  Stream 1: SMTP domains (responsive + catch-all + no_mx)
    - 50 workers
    - Batched SMTP connections (one connection per domain, 8 RCPT TO + 1 fake)
    - Chunks of 500 contacts

  Stream 2: M365/gateway domains
    - 50 workers
    - HTTPS calls to Microsoft CDN
    - Chunks of 500 contacts
```

**Why two streams:** SMTP checks take 1-3 seconds. M365 HTTPS calls take 3-8 seconds. Mixing them in one thread pool means slow M365 calls block fast SMTP workers. Splitting doubles throughput.

**Upload and run pattern:**
```bash
scp script.py root@YOUR_VPS_IP:/root/
scp data.json root@YOUR_VPS_IP:/root/
# Write a launcher.sh, upload it, then execute:
ssh root@YOUR_VPS_IP "bash /root/launcher.sh"
```

Note: SSH with `&` backgrounding can fail due to shell escaping. Always use a launcher script.

---

## Section 3b: Execution Flow (5 Steps)

Every email finding run follows this exact sequence. Steps 3a and 3b run in parallel. All other steps are sequential. Failure in one step does not require restarting the entire pipeline.

### Step 0: Pre-filter (instant, zero API calls)

Input: contacts with first_name, last_name, domain

Actions:
- Check domain against disposable blocklist → skip
- Skip contacts with empty first_name, last_name, or domain
- Extract unique domains from remaining contacts

Output: filtered contact list + unique domain list

**Failure mode:** None. Pure local computation. Cannot fail.

### Step 1: Domain classification on VPS (25 workers, ~4 min per 3,000 domains)

Input: unique domain list

Actions:
- Upload domain list to VPS
- Run classification script with 25 workers
- One fake SMTP probe per domain via yourdomain.com (HELO yourdomain.com, MAIL FROM verify@yourdomain.com)

Output per domain: responsive / catch_all / no_mx / m365 / gateway

Cache: save to domain_classification.json after every 20-domain chunk

**Failure modes:**
- VPS unreachable → Check Hetzner console. Reboot or redeploy (Section 8). Do NOT fall back to local execution.
- SMTP verifier down → SSH in: `systemctl restart smtp-verifier && docker restart reacher`. Re-run. Checkpoint resumes automatically.
- Partial completion → Re-run same command. Already-classified domains skipped via cache.

**Timeout:** If Step 1 exceeds 15 min for 3,000 domains, check `ss -tnp | grep python3 | wc -l` on VPS. If 0 connections, process died. Kill and restart.

### Step 2: Route contacts (instant, zero API calls)

Input: contacts + domain classifications

Actions:
- no_mx contacts → mark skip, done
- responsive + catch_all contacts → SMTP stream
- m365 + gateway contacts → M365 stream

Output: two contact lists for parallel processing

**Failure mode:** None. Pure local computation.

**Sanity check:** If more than 40% of domains are no_mx, DNS resolution is broken on VPS. Fix: `echo "nameserver 8.8.8.8" > /etc/resolv.conf`

### Step 3: Find emails (two parallel streams on VPS, ~20 min per 10,000 contacts)

**Stream A: SMTP (50 workers)**
- Domains: responsive + catch_all
- Per contact: 8 RCPT TO patterns + 1 fake probe in one SMTP session via yourdomain.com
- Greylisting: retry 450/451 once after 5s
- Results: 250 + fake rejected → high | 250 + fake accepted → low (catch-all) | all rejected → none

**Stream B: M365 (50 workers)**
- Domains: m365 + gateway
- Per contact: realm check (cached per domain), then up to 4 patterns via GetCredentialType
- Results: IfExistsResult == 0 → medium | not managed or not found → none

**Both streams run simultaneously on VPS. Chunks of 500. Incremental save after each chunk.**

**Failure modes:**
- SMTP stream stalls (no progress 5+ min) → Check connections on VPS. If active but no completions, a target server is hanging. Kill and re-run. Checkpoint resumes.
- M365 rate limited (429) → Reduce workers to 20, add 1s delay. Retry.
- M365 completely down → Skip M365 stream. Those contacts get "none." Re-run M365 stream later.
- One stream done, other stuck → Save finished stream results. Kill stuck stream. Re-run only the stuck stream. Merge in Step 4.

**Timeout:** SMTP stream should process at 1,500/min minimum. If below 500/min, port 25 may be throttled. Wait 1 hour or switch VPS.

**Circuit breaker:** If same domain fails 3 times across different contacts, blacklist that domain for rest of run.

### Step 4: Merge + apply (instant)

Input: results from both streams

Actions:
- Combine all found emails with confidence scores
- high: SMTP verified on responsive server (fake probe rejected)
- medium: M365 identity confirmed
- low: catch-all domain, first.last@ assigned, deliverable but pattern uncertain
- none: not found, dead domain, disposable, gateway blocked
- **All tiers (high, medium, low) are deliverable and usable for outreach**
- Write to CSV with confidence column

**Failure mode:**
- Result files missing → Check /root/ on VPS for partial files. Download and merge manually.
- CSV write fails → Save to JSON first as backup. Apply to CSV separately.

### Overall rules
- Never re-run a completed stream. Only re-run failed or incomplete streams.
- Never run Step 3 without Step 1 completed.
- If total runtime exceeds 45 min for 10,000 contacts, diagnose before continuing.
- Always check VPS health (Section 1) before starting any run.
- Monitor VPS during run: `tail -f /root/*.log`

---

## Section 4: Domain Classification (Step Zero)

Before trying real email patterns, classify each unique domain. For bulk batches, do this ON the VPS with 25 workers.

### Inline Classification (preferred for bulk)

Instead of a separate Step Zero pass, include the fake probe IN the same SMTP session as real patterns:

```python
def smtp_session_verify(emails, mx_host):
    fake = f"xyzfake98765zzz@{emails[0].split('@')[1]}"
    all_to_test = emails + [fake]
    results = {}
    fake_code = None
    with smtplib.SMTP(mx_host, 25, timeout=10) as s:
        s.ehlo(HELO)
        s.mail(FROM_EMAIL)
        for email in all_to_test:
            code, _ = s.rcpt(email)
            # Greylisting: 450/451 = temporary rejection, retry once after 5s
            if code in (450, 451):
                time.sleep(5)
                try:
                    code, _ = s.rcpt(email)
                except:
                    code = 0
            if email == fake:
                fake_code = code
            else:
                results[email] = code
    is_catch_all = (fake_code == 250)
    return results, is_catch_all
```

One TCP connection does classification AND verification. The fake probe tells you if the server discriminates (responsive) or accepts everything (catch-all).

### Separate Classification (for domain pre-scan)

When you need to classify domains before deciding how to process contacts:

Upload domain list to VPS, run with 25 workers:
```python
def classify(domain):
    mx = get_mx(domain)
    if not mx: return "no_mx"
    code = smtp_probe(f"xyzfake12345@{domain}", mx)
    if code == 250: return "catch_all"
    if code in (550, 551, 553): return "responsive"
    if is_m365_managed(domain): return "m365"
    return "gateway"
```

| Classification | Meaning | Action |
|----------------|---------|--------|
| responsive | Server validates emails | SMTP batched patterns + fake probe |
| catch_all | Accepts everything | Assign first.last@, confidence: low |
| no_mx | Dead domain | Skip |
| m365 | Microsoft 365 managed | M365 identity check on top 4 patterns |
| gateway | Blocked + not M365 | Mark unverifiable |

---

## Section 5: Waterfall Logic

### SMTP Stream (responsive + catch-all domains)

```python
# One SMTP session per contact: 8 patterns + 1 fake probe
emails = [pattern(first, last) + "@" + domain for pattern in PATTERNS]
results, is_catch_all = smtp_session_verify(emails, mx_host)

if not is_catch_all:
    # Responsive server. Trust 250s.
    for email in emails:
        if results[email] == 250:
            return email, "smtp_verified", "high"
    return None, "not_found", "none"
else:
    # Catch-all. first.last@ is best guess.
    return f"{first}.{last}@{domain}", "catch_all", "low"
```

8 email patterns (in priority order):
1. first.last@domain   (~50%)
2. flast@domain         (~20%)
3. first@domain         (~8%)
4. firstlast@domain     (~6%)
5. first_last@domain    (~5%)
6. last.first@domain    (~4%)
7. firstl@domain        (~3%)
8. f.last@domain        (~2%)

### M365 Stream (m365 + gateway domains)

```python
if is_m365_managed(domain):
    for pattern in PATTERNS[:4]:  # top 4 only
        email = pattern(first, last) + "@" + domain
        if m365_user_exists(email):
            return email, "m365_verified", "medium"
    return None, "not_found_m365", "none"
else:
    return None, "not_m365", "none"
```

---

## Section 6: Confidence Scoring (MANDATORY)

Every found email MUST have a confidence level. Never return an email without one.

| Confidence | Meaning | How verified | Reliability |
|-----------|---------|-------------|-------------|
| **high** | SMTP 250 on responsive server (fake probe rejected) | Mail server confirmed mailbox exists AND rejects fakes | 95%+ |
| **medium** | M365 identity confirmed | Microsoft Azure AD confirms user account exists | 85% |
| **low** | Catch-all domain, pattern is a guess | Domain accepts all emails, first.last@ is industry default | 50% deliverable, pattern uncertain |
| **none** | Not found | All methods failed | N/A |

**Rules:**
- Never trust M365 alone without attempting SMTP first. M365 confirms identity, not deliverability.
- Never assign an email to a catch-all domain without marking it "low" confidence.
- Never count "low" confidence emails as "found" in success metrics without disclosure.

---

## Section 7: Processing Rules

### Chunked Futures (CRITICAL)

Never submit all contacts as futures at once. Process in chunks of 500.

```python
for i in range(0, len(contacts), 500):
    chunk = contacts[i:i+500]
    with ThreadPoolExecutor(max_workers=50) as ex:
        results = list(ex.map(process_fn, chunk))
    save_incrementally(results)
    print_progress()
```

Submitting 10,000+ futures at once causes thread pool exhaustion and hangs with no progress output.

### Retry Logic

- Max 2 retries per SMTP connection failure
- 3-second delay between retries (NOT 5, causes cascading slowdowns)
- Retry on: socket.timeout, ConnectionRefusedError, OSError
- Do NOT retry on: 550/551/553 (definitive rejection)

### Greylisting Handling

Some mail servers use greylisting: they reject the first SMTP attempt with a 450 or 451 temporary code, then accept on retry after a delay. This is a deliberate anti-spam measure, not a failure.

```python
# Inside the SMTP session, after smtp.rcpt(email):
if code in (450, 451):
    # Greylisting detected. Wait and retry in same session.
    time.sleep(5)
    try:
        code, _ = smtp.rcpt(email)  # retry same RCPT TO
    except:
        code = 0  # connection dropped, move on
```

- Only retry 450/451 codes, not 550/553 (those are permanent rejections)
- Retry once per greylisted RCPT TO, not per connection
- 5-second wait is sufficient. Most greylisting servers accept on second attempt within 5-30 seconds.
- This recovers 3-5% of contacts that would otherwise show as "not_found"

### Disposable Email Domain Blocklist

Before attempting ANY verification, check if the domain is a known disposable/throwaway email provider. These domains accept all emails but the mailboxes are temporary and worthless for outreach.

```python
DISPOSABLE_DOMAINS = {
    'mailinator.com', 'guerrillamail.com', 'tempmail.com', 'throwaway.email',
    'yopmail.com', 'sharklasers.com', 'guerrillamail.info', 'grr.la',
    'guerrillamail.biz', 'guerrillamail.de', 'guerrillamail.net',
    'trash-mail.com', 'trashmail.com', 'trashmail.me', 'trashmail.net',
    'dispostable.com', 'maildrop.cc', 'mailnesia.com', 'tempr.email',
    'temp-mail.org', 'fakeinbox.com', 'mohmal.com', 'getnada.com',
    'emailondeck.com', 'mintemail.com', 'mytemp.email',
}

def is_disposable(domain):
    return domain.lower() in DISPOSABLE_DOMAINS
```

- Zero API calls. Pure set lookup. Microseconds per check.
- Check BEFORE Step Zero. No point classifying or probing a disposable domain.
- If disposable, mark as `status: "disposable", confidence: "none"` and skip.
- Maintain the set. Add domains as you encounter them.

### Caching

MX record resolution via `dns.resolver` takes 100-500ms per lookup. For 4,500 domains with 16,000 contacts, uncached DNS adds 30-75 minutes of pure wait time.

```python
mx_cache = {}

def get_mx(domain):
    if domain in mx_cache:
        return mx_cache[domain]
    try:
        answers = dns.resolver.resolve(domain, 'MX')
        mx = sorted(answers, key=lambda x: x.preference)[0].exchange.to_text()
    except:
        mx = None
    mx_cache[domain] = mx  # cache even None results to avoid re-resolving dead domains
    return mx
```

Cache rules:
- Cache MX records per domain in a dict. Persist across all contacts in the same run.
- Cache M365 realm check per domain (one HTTPS call, reuse for all contacts at that domain)
- Cache None/failure results too. A domain with no MX won't grow one mid-run.
- Save results incrementally after each chunk (crash recovery)

---

## Section 8: Infrastructure Setup (first time or after teardown)

1. **Create Hetzner CX23** ($4.91/mo, Helsinki or Nuremberg, Ubuntu 24.04)

2. **SSH in**: `ssh root@{IP}`

3. **Request port 25 unblock** via Hetzner support ticket

4. **Install Docker + Reacher + dnspython**:

```bash
apt update && apt install -y docker.io python3-dnspython
docker run -d --name reacher --restart always \
  -e RCH__FROM_EMAIL=verify@yourdomain.com \
  -e RCH__HELLO_NAME=mail1.yourdomain.com \
  -e RCH__HTTP_HOST=0.0.0.0 \
  -p 8080:8080 reacherhq/backend:latest
```

5. **Deploy SMTP Verifier**:
   - Upload `smtp_api.py` to `/root/smtp_api.py`
   - Create systemd service at `/etc/systemd/system/smtp-verifier.service`:

```ini
[Unit]
Description=SMTP Email Verifier API
After=network.target

[Service]
ExecStart=/usr/bin/python3 /root/smtp_api.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

   - Enable and start:

```bash
systemctl daemon-reload && systemctl enable smtp-verifier && systemctl start smtp-verifier
```

6. **Open port 8081**: `ufw allow 8081/tcp`

7. **DNS (yourdomain.com on Cloudflare)**:
   - A record: `mail1.yourdomain.com` -> server IP
   - SPF: `v=spf1 ip4:{server_ip} ~all`
   - rDNS in Hetzner: server IP -> `mail1.yourdomain.com`

---

## Section 9: Rules (non-negotiable)

1. **Never run SMTP verification locally.** Always through the VPS. Port 25 is blocked on home/office networks. Running locally results in 30-40% false `no_smtp` failures.

2. **Never use row index as matching key.** Always use LinkedIn URL or another stable unique identifier.

3. **Always use `yourdomain.com` as sender domain** (from_email and HELO). It has proper DNS, SPF, and rDNS configured.

4. **Always include a fake probe in SMTP sessions.** Never trust a 250 response without confirming the server rejects fakes. A 250 on a catch-all server means nothing.

5. **Always assign confidence scores.** Every found email gets high/medium/low. No unscored results.

6. **Always run bulk operations on the VPS directly.** Never route 100+ SMTP checks through the HTTP API from the client. Upload a script, run it on the server.

7. **Always split SMTP and M365 into parallel streams.** Never mix them in the same thread pool. Different speed profiles cause mutual blocking.

8. **Always process in chunks of 500.** Never submit all futures at once. Chunked processing gives incremental progress and prevents hangs.

9. **Cache domain classification and MX records.** Never re-check a domain already classified. Never re-resolve MX for a domain already looked up.

10. **Never trust M365 identity check alone.** It confirms the user exists in Azure AD, not that the mailbox accepts email. Always attempt SMTP verification first, use M365 as fallback for gateway-blocked domains only.

11. **Always check disposable domains before verification.** Check against the blocklist BEFORE Step Zero. No point probing a throwaway domain. Zero API calls, pure set lookup.

---

## Section 10: Performance Benchmarks (measured, not estimated)

| Method | Rate | Use case |
|--------|------|----------|
| HTTP API from client (3 workers) | 12/min | Small batches only |
| VPS-side smtplib (50 workers, batched) | 1,500/min | Bulk SMTP verification |
| VPS-side domain classification (25 workers) | 700/min | Step Zero pre-scan |
| M365 stream (50 workers) | 1,300/min | Gateway/M365 domains |
| Combined two-stream (SMTP + M365 parallel) | 16,000 contacts in 30 min | Full pipeline |
| Full pipeline (10,000 contacts) | ~23 min | Steps 0-4 end to end |
| Expected deliverable rate | ~31% | high 10% + medium 8% + low 13% |

These benchmarks are from processing 16,148 contacts across 4,503 domains on a Hetzner CX23 (2 vCPU, 4GB RAM). VPS CPU never exceeded 20%.
