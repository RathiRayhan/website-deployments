# Standard Email Infrastructure & Deliverability Runbook

## Overview
This runbook outlines the standard operating procedure (SOP) for configuring a professional custom domain email infrastructure. It covers DNS authentication (SPF, DKIM, DMARC), email deliverability optimization, warmup strategies, and receive-only routing architecture using Cloudflare.

---

## Phase 1: Primary Email Setup (Zoho Mail)
When a full-fledged email hosting (sending and receiving) is required, Zoho Mail (or Google Workspace) is the standard choice.

### 1. Account & Domain Verification
- Create a Business account on Zoho Mail.
- Add the custom domain.
- Verify domain ownership via a standard TXT record or CNAME provided by the host.

### 2. Core DNS Configuration (Cloudflare / DNS Provider)
Once verified, configure the inbound mail servers.
- **MX Records:** Add Zoho's MX records to route incoming emails to Zoho's servers.
    - `mx.zoho.com` (Priority 10)
    - `mx2.zoho.com` (Priority 20)
    - `mx3.zoho.com` (Priority 50)

---

## Phase 2: Domain Authentication (SPF, DKIM, DMARC)
Authentication is strictly required to prevent spoofing and ensure emails land in the inbox instead of the spam folder.

### 1. SPF (Sender Policy Framework)
Defines which IP addresses and services are authorized to send emails on behalf of the domain.
- **Type:** `TXT`
- **Name:** `@`
- **Value:** `v=spf1 include:zohomail.com ~all`
- **Production Note:** Always use `~all` (Softfail) initially. If the client uses third-party marketing tools (Mailchimp, Brevo, Google Workspace), they MUST be included in the SPF record before the `~all` mechanism (e.g., `v=spf1 include:zohomail.com include:servers.mcsv.net ~all`).

### 2. DKIM (DomainKeys Identified Mail)
Adds a cryptographic signature to outgoing emails.
- Generate the DKIM record from the Zoho Admin panel.
- Add it as a `TXT` record in the DNS (e.g., `zmail._domainkey`).

### 3. DMARC (Domain-based Message Authentication, Reporting, and Conformance)
Email providers do not generate this by default; it must be created manually.
- **Type:** `TXT`
- **Name:** `_dmarc`
- **Value:** `v=DMARC1; p=none; rua=mailto:dmarc-reports@yourdomain.com;`
- **Policy Lifecycle:**
    1. **Monitoring (`p=none`):** Start here. It allows all emails to deliver but sends XML reports to the `rua` email.
    2. **Analysis:** Analyze reports using tools like **Cloudflare DMARC Management** or **Postmark**.
    3. **Enforcement (`p=quarantine` or `p=reject`):** Once all legitimate sending sources are verified and included in the SPF, upgrade the policy to block spoofing attacks.

---

## Phase 3: Deliverability Testing & Warmup
Even with a 10/10 technical setup, emails may land in spam due to domain reputation.

### 1. Diagnostics & Scoring
- Use **MXToolbox** to verify DNS propagation.
- Use **Mail-Tester.com** to check the spam score. 
- *Note:* A perfect configuration yields a 10/10 score. However, untrustworthy or cheap TLDs (e.g., `.xyz`, `.top`) may incur a penalty (e.g., -1.5 points).

### 2. The Warmup Process
New domains have zero sending reputation. Google and Microsoft AI will flag sudden traffic.
- Send test emails to various accounts.
- If they land in spam, manually mark them as **"Report not spam"** or **"Looks safe"**.
- Reply to the emails naturally. Repeat this for 1-2 weeks to build a solid IP/Domain reputation.

---

## Phase 4: Receive-Only Architecture (Cloudflare Email Routing)
If the client only needs to receive emails (e.g., website contact forms) without paying for premium hosting, use Cloudflare Email Routing.

### 1. Configuration
- Ensure all existing MX records (e.g., Zoho) are deleted.
- Enable Email Routing in Cloudflare.
- Add Cloudflare's MX and SPF (`v=spf1 include:_spf.mx.cloudflare.net ~all`) records.
- Create a Route: `admin@yourdomain.com` -> Forwards to -> `personal.email@gmail.com`.

### 2. The Forwarding Limitation (SPF Alignment & ARC)
- **The Problem:** Forwarding breaks SPF alignment because the email is delivered by Cloudflare's IPs, not the original sender's IPs.
- **The Backend Fix:** Cloudflare automatically applies **SRS (Sender Rewriting Scheme)** and **ARC (Authenticated Received Chain)** headers to tell Gmail it's a legitimate forward.
- **The Spam Reality:** Despite ARC/SRS, strict providers like Google will often throw forwarded emails into spam, especially if the domain is new or uses a low-tier TLD.
- **The Ultimate Bypass:** Inside the destination Gmail account, create a filter:
    - **Condition:** `To: admin@yourdomain.com`
    - **Action:** `Never send it to Spam`.

### 3. Outbound Sending Alternatives (SMTP)
Cloudflare does not support sending emails. 
- *Legacy Method:* Using Gmail App Passwords to "Send As" a custom domain is being deprecated (2027).
- *Modern Solution:* Use third-party transactional SMTP servers like **SMTP2GO**, **Brevo**, or **Postmark**. 
- *Requirement:* You must include their specific values in your domain's SPF and DKIM records to prevent outbound mails from bouncing.

---

## Phase 5: Production Handoff & Compliance

### 1. Data Privacy & Reporting
Ensure DMARC aggregate reports (`rua`) are routed exclusively to the client's internal administrative email (e.g., `admin@clientdomain.com`). Never route client telemetry or mail logs to personal, external, or unauthorized monitoring addresses to maintain strict data privacy.

### 2. Monitoring Infrastructure (Value-Added)
To simplify XML report analysis, recommend or integrate third-party DMARC visualization dashboards (e.g., Cloudflare DMARC Management, Postmark). This provides the client with an accessible, graphical overview of their domain's email health.

### 3. Fail-Safe Deployment Strategy
Always execute the initial delivery with non-disruptive policies:
- **SPF:** `~all` (Softfail)
- **DMARC:** `p=none` (Monitoring mode)
This fail-safe configuration prevents accidental blockages of the client's undocumented third-party sending services (e.g., CRMs, Mailchimp, automated website forms).

### 4. Policy Enforcement Roadmap
Provide the client with clear, documented instructions advising them to upgrade their DMARC policy to `p=quarantine` or `p=reject` only after a standard 2-4 week monitoring phase confirms no legitimate traffic is failing alignment.
