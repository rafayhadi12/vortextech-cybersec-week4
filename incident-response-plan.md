# Mini Incident Response Plan
### Scenario: Customer Database Exposure via Unsecured API Endpoint
**Company:** BrightCart (fictional e-commerce platform)
**Framework used:** NIST Incident Response Lifecycle (SP 800-61)

---

## 1. Breach Scenario Overview

BrightCart operates an online retail platform serving roughly 250,000 registered customers. During a routine backend update, a developer deployed a new internal API endpoint (`/api/v2/customer-lookup`) intended for the internal support dashboard. The endpoint was accidentally left publicly accessible with no authentication check and returned full customer records — including names, emails, shipping addresses, and hashed passwords — when queried with a sequential customer ID.

A security researcher discovered the exposed endpoint while performing routine internet-wide scanning for misconfigured APIs and reported it to BrightCart's security email as part of a responsible disclosure. Logs later confirmed the endpoint had also been queried by at least two automated scraping bots in the 96 hours before the report, indicating the data had likely already been harvested by unknown third parties.

**Data at risk:** ~250,000 customer records (name, email, address, hashed password, order history metadata).

---

## 2. Preparation

What should already have been in place before this incident:

- A documented Incident Response Plan and a named Incident Response Team (IRT) with defined roles (Incident Commander, Security Analyst, Comms Lead, Legal Liaison).
- API gateway configured to enforce authentication/authorization on all endpoints by default (deny-by-default policy).
- Automated pre-deployment security scanning (SAST/DAST) integrated into the CI/CD pipeline to catch unauthenticated endpoints before release.
- Centralized logging and a SIEM (e.g., Splunk, ELK) with alerting rules for anomalous API traffic (e.g., high-volume sequential ID queries).
- Regular backups of customer databases, tested for integrity and restorability.
- A responsible disclosure / bug bounty channel (security@brightcart.example) so researchers have a clear reporting path — which is in fact how this breach was caught.

---

## 3. Detection & Analysis

**How it was discovered:** An external security researcher reported the exposed endpoint via the responsible disclosure channel.

**Initial investigation steps:**
1. Security team validates the report by attempting the same query in a controlled test environment.
2. Pull API gateway and application logs for `/api/v2/customer-lookup` to determine:
   - When the endpoint went live.
   - Total number of requests and unique source IPs.
   - Whether request patterns look like automated scraping (sequential IDs, high request rate, no session cookies).
3. Cross-reference source IPs against known scanner/bot IP ranges and threat intel feeds.
4. Classify severity: confirmed unauthenticated access to PII at scale → **Critical (Sev-1)**.
5. Formally declare an incident and activate the IRT.

---

## 4. Containment

**Short-term containment (within the first hour):**
- Immediately disable or restrict the `/api/v2/customer-lookup` endpoint at the API gateway (block public access).
- Rotate any API keys or internal service tokens associated with the affected microservice, in case credentials were also exposed.
- Enable enhanced logging/rate-limiting on adjacent endpoints in case the misconfiguration pattern was repeated elsewhere.

**Long-term containment:**
- Deploy a hotfix that adds an authentication middleware requirement to the endpoint (and audit all other endpoints in the same service for the same gap).
- Block identified scraper IP ranges at the WAF level.

---

## 5. Eradication

- Conduct a full audit of all API endpoints across the platform to confirm no other routes have the same missing-authentication misconfiguration.
- Patch the root cause: enforce that new endpoints inherit a default "require-auth" middleware unless explicitly and reviewably marked public.
- Revoke and reissue any credentials/tokens that could have been harvested alongside customer data.
- Remove any unauthorized access if evidence shows an attacker (not just the researcher) established persistence — in this scenario, no server compromise occurred (it was a data-exposure issue, not a server breach), so this step focuses on confirming no backdoors were planted via the exposed data (e.g., credential stuffing attempts using the hashed passwords).

---

## 6. Recovery

- Re-enable the corrected endpoint behind authentication and monitor its traffic closely for 2 weeks post-fix.
- Force a password reset for all affected accounts as a precaution, since password hashes were exposed.
- Monitor for credential-stuffing attempts on the login page using the leaked email/password combinations (rate-limit and flag anomalies).
- Validate system integrity from backups if any tampering is suspected (in this case, read-only exposure, so primary focus is monitoring rather than restoration).
- Confirm with the CI/CD team that the new deploy-time authentication check is active before closing the incident.

---

## 7. Lessons Learned (Post-Incident Review)

Conducted within 1–2 weeks of resolution. Likely findings:

- The CI/CD pipeline lacked an automated check for unauthenticated public endpoints — this must be added as a hard gate before deploy.
- No internal alerting existed for high-volume sequential-ID API queries, which is a classic enumeration pattern; SIEM rules need updating.
- The responsible disclosure channel worked as intended and should be publicized more (bug bounty program considered).
- Update the IR plan itself with the actual timeline from this incident to refine future response time targets.

---

## 8. Internal Communication Plan

| When | Who is notified | By whom |
|---|---|---|
| Immediately (0–1 hr) | Incident Commander, Security Team, Engineering Lead | Analyst who confirmed the report |
| Within 2–4 hrs | Executive leadership (CTO/CEO), Legal/Compliance team | Incident Commander |
| Within 24 hrs | All engineering staff (need-to-know basis), PR/Comms team briefed | Incident Commander + Comms Lead |
| Within legally mandated timeframe (varies by jurisdiction/data protection law) | Affected customers, relevant regulators | Legal/Compliance team + Comms Lead |
| Post-resolution | Full internal post-mortem shared org-wide | Incident Commander |

*Note: exact regulatory notification deadlines depend on applicable data protection law (e.g., GDPR's 72-hour rule) and are handled by Legal — not detailed here beyond the conceptual timeline.*

---

## 9. Preventative Measures

1. **Enforce deny-by-default authentication** on all API endpoints via the API gateway, with automated CI/CD checks that block deployment of any endpoint missing an auth requirement.
2. **Add anomaly-detection alerting** for sequential-ID enumeration and high-volume unauthenticated queries in the SIEM.
3. **Formalize the responsible disclosure program** (with a clear security.txt / bug bounty page) so future researchers have an even faster, more visible reporting path — reducing time-to-detection.
