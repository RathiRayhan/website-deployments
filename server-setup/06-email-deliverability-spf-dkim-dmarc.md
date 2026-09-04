# Email Authentication & Deliverability Architecture
**Objective:** Secure outbound email communication, prevent domain spoofing, and ensure high deliverability using SPF, DKIM, and DMARC.

---

## 1. Introduction
When configuring an email infrastructure (e.g., `user@example.com`), organizations typically rely on either self-hosted SMTP servers or managed Email Service Providers (ESPs) like Google Workspace, Zoho, or Amazon SES. 

To prevent domain spoofing and ensure legitimate emails reach the inbox, DNS-based authentication mechanisms must be strictly configured. The three pillars of email security are **SPF, DKIM, and DMARC**.

---

## 2. SPF (Sender Policy Framework)
SPF is a DNS `TXT` record that explicitly lists the authorized IP addresses or hostnames allowed to send emails on behalf of a domain.

### How it works:
During the initial SMTP connection (`MAIL FROM` phase), the receiving server checks the `Return-Path` domain. It then queries the DNS of that domain to verify if the sending IP is authorized in the SPF record.

### Qualifiers (Prefixes)
Mechanisms in an SPF record are prefixed with a qualifier to dictate the policy:
* `+` **Pass (Default):** The IP is authorized. (e.g., `+a` is the same as `a`).
* `-` **Fail (Hard Fail):** The IP is strictly NOT authorized; reject the email.
* `~` **SoftFail:** The IP is not authorized, but accept the email and mark it as suspicious/spam. (Most common in production).
* `?` **Neutral:** No specific policy; treat it as if no SPF record exists.

### Common Mechanisms
* `all` : Placed at the very end of the record to apply a default policy to any IP not matched by prior mechanisms.
* `ip4` / `ip6` : Authorizes specific IPv4 or IPv6 addresses or CIDR ranges.
* `a` : Authorizes the IP address(es) found in the domain's A record.
* `mx` : Authorizes the IP address(es) of the domain's MX (Mail Exchange) servers.
* `include` : Authorizes the servers of a third-party ESP (e.g., `include:spf.zoho.com`).

### Deprecated & Advanced Mechanisms
* `ptr` : **(Do not use)** Validates via reverse DNS. Deprecated (RFC 7208) as it heavily burdens DNS servers and causes timeouts.
* `exists` : **(Advanced)** Resolves any domain name to check if an A record exists. Used primarily by large enterprises to build dynamic SPF macros to bypass the strict 10 DNS lookup limit.

### Production-Grade SPF Examples
**Example 1: Standard Enterprise Setup**
```text
v=spf1 a mx include:_spf.google.com include:servers.mcsv.net ~all
```
*Explanation:* Allows the root domain's A and MX records to send mail. Also authorizes Google Workspace and Mailchimp (`mcsv.net`). All other unauthorized IPs will SoftFail (`~all`). 
*(Note: Always keep total DNS lookups under 10 to prevent SPF failure).*

**Example 2: Specific IP Ranges**
```text
v=spf1 ip4:192.168.0.1/16 ~all
```
*Explanation:* Allows any IP address within the `192.168.0.1` to `192.168.255.255` subnet to send mail. Others will SoftFail.

**Example 3: Defensive Record (No Email Allowed)**
```text
v=spf1 -all
```
*Explanation:* Used for parked domains or domains that should NEVER send emails. Rejects all outbound mail attempts.

---

## 3. DKIM (DomainKeys Identified Mail)
DKIM ensures message integrity and authenticity by applying a cryptographic signature to the email. 

### How it works:
The sending server signs the email using a **Private Key**. The receiving server retrieves the corresponding **Public Key** from the sender domain's DNS (`TXT` or `CNAME` record) and uses it to verify the signature. If the email was tampered with during transit, the decryption fails, and DKIM fails.

### Obtaining the DKIM Record:
You do not write this record manually. The ESP (e.g., Zoho, AWS SES) generates a public key block in their dashboard. You simply copy and paste it into your DNS management portal as a `TXT` or `CNAME` record.

---

## 4. DMARC (Domain-based Message Authentication, Reporting, and Conformance)
DMARC is the policy and reporting layer. It ties SPF and DKIM together to enforce **Identity Alignment**.

### The Concept of Alignment:
Neither SPF nor DKIM natively verifies the visual `From` header (what the end-user sees). 
* SPF only checks the hidden `Return-Path` domain.
* DKIM only checks the domain that signed the email.
DMARC mandates that the domain in the visual `From` header **MUST MATCH** either the SPF domain or the DKIM signature domain. If alignment fails, DMARC instructs the receiver on how to handle the email based on your defined policy.

### DMARC Record Syntax
```text
v=DMARC1; p=quarantine; rua=mailto:reports@example.com;
```
* `v=DMARC1` : The protocol version.
* `p` (Policy) : Can be `none` (monitor only), `quarantine` (send to spam), or `reject` (block completely).
* `rua` : The email address where receiving servers will send XML-based aggregate reports regarding passing/failing emails. Often, organizations use specialized reporting services (like Dmarcian or Postmark) to parse these XML files.

### Third-Party ESP Integration (Vendor Management)
An organization often sends mail from multiple sources (Google, Mailchimp, Zendesk). DMARC is a domain-wide policy. Therefore, as an infrastructure engineer, you must individually configure SPF and DKIM alignment inside the dashboard of *every single ESP* used by the organization. Failing to align even one ESP will cause their emails to be blocked by DMARC.

---

## 5. Architectural FAQs & Edge Cases

**Q: Why doesn't SPF just verify the visual `From` address?**
**A:** This is a limitation of the SMTP protocol architecture. SPF validates at the `MAIL FROM` phase (envelope layer), which only contains the `Return-Path`. The visual `From` address is transmitted later during the `DATA` phase. SPF was designed to drop unauthorized connections early, before processing the data body.

**Q: Why doesn't DKIM natively force alignment with the `From` address?**
**A:** To preserve the functionality of third-party ESPs (like Mailchimp) and legacy systems. If DKIM enforced strict alignment natively, ESPs could not sign emails on behalf of their users without highly complex setups, breaking a massive portion of legitimate internet email traffic.

**Q: If DMARC requires strict alignment, won't Group Emails / Mailing Lists fail?**
**A:** Yes. When a mailing list receives and forwards an email, it modifies the body (breaking the DKIM signature) and changes the `Return-Path` (breaking SPF). Consequently, DMARC fails. 
*Solutions:* Modern mailing list servers mitigate this by using **From Address Rewriting** (modifying the `From` header to match the list's domain) or by utilizing **ARC (Authenticated Received Chain)**, a protocol that vouches for the email's original authentication status before it was modified.
