# Site Audit — sicarautos.com.mx

**Date:** 2026-08-18 (UTC)
**Context:** DNS for the domain was recently changed. This audit focuses on verifying the DNS migration and everything reachable from this environment.

> **Scope limitation:** This audit environment runs behind a restricted network egress policy that does not include `sicarautos.com.mx` in its allowlist. DNS-layer checks (records, propagation, authoritative zone) completed fully; TCP/TLS reachability on port 443 was confirmed; but HTTP *content* checks (redirects, page content, security headers, SEO on-page, performance) were blocked by the egress gateway. A follow-up content audit requires adding `sicarautos.com.mx` and `www.sicarautos.com.mx` to the session's network egress allowlist.

---

## Executive summary

The DNS migration to **Vercel DNS** is complete and fully propagated — but the zone is **web-only**. There are **no MX, SPF, or DMARC records**, which means:

1. 🔴 **If this domain previously received email (e.g. `contacto@sicarautos.com.mx`), that email is now bouncing.** This is the single most urgent item to verify.
2. 🟠 With no SPF/DMARC, anyone can spoof email from `@sicarautos.com.mx`, and any mail the business sends "from" the domain is likely to land in spam.

The web side of the migration looks healthy: nameservers, A records, propagation, CAA records, and TLS reachability are all consistent and correct for a Vercel-hosted site.

---

## 1. DNS migration status — ✅ Complete and propagated

| Item | Value | Assessment |
|---|---|---|
| Nameservers | `ns1.vercel-dns.com`, `ns2.vercel-dns.com` | Vercel DNS (NS1-backed) |
| Apex A records | Multiple IPs in `216.150.0.0/16` | Vercel's current anycast edge range ✅ |
| `www` | A records in same Vercel range | ✅ |
| A-record TTL | 1800s (30 min) | Reasonable post-migration value |
| SOA serial | `1787085830` = **2026-08-18 20:43:50 UTC** | Zone was updated ~1.5 h before this audit — confirms the recent change |
| AAAA (IPv6) | None | Informational — Vercel serves v4 only here |

**Propagation check** — apex and `www` A records queried against four independent resolvers (Google 8.8.8.8, Cloudflare 1.1.1.1, Quad9 9.9.9.9, and the local resolver): **all four return Vercel IPs**. No resolver is still serving the old host. Propagation is effectively complete (rotating IP sets across queries are normal Vercel anycast behavior, not an inconsistency).

## 2. Email — 🔴 Critical gaps

Queried against both public resolvers and the authoritative Vercel nameservers:

| Record | Status | Impact |
|---|---|---|
| **MX** | **None** | The domain **cannot receive email**. If mailboxes like `ventas@` / `contacto@sicarautos.com.mx` existed on the previous host (the old setup appears to have been cPanel-style hosting), senders are now getting bounces — for a dealership this can mean **lost sales leads**. |
| **SPF** (TXT) | None | No sender authorization policy. |
| **DMARC** (`_dmarc` TXT) | None | No spoofing protection or reporting. |
| **DKIM** (common selectors) | None found | No signing keys published. |

**Follow-up research (public sources):** the dealership's *advertised* contact email is `ventas1@autosicar.com.mx` — a **sister domain** (`autosicar.com.mx`) that is unaffected by this migration: it still runs on Triara (Telmex) DNS with working MX records pointing at `maila`/`mailb.exchangeadministrado.com` (managed Exchange) and a valid SPF record. So the main sales inbox is likely fine. However, at least one staff mailbox on `@sicarautos.com.mx` (Carlos Ivan Maldonado, listed in a dealer directory) appears in public listings — if mailboxes on *this* domain were active, they are bouncing now.

**Action required:**
- Confirm with the business whether any `@sicarautos.com.mx` mailboxes are active. If yes, restore the MX records in the Vercel DNS dashboard **immediately** — if they lived on the same managed-Exchange service as the sister domain, that is `MX 10 maila.exchangeadministrado.com` / `MX 10 mailb.exchangeadministrado.com` plus TXT `v=spf1 include:spf.exchangeadministrado.com ~all`, then DKIM/DMARC.
- If the domain **never sends or receives email**: publish a null policy to block spoofing:
  - TXT `@` → `v=spf1 -all`
  - TXT `_dmarc` → `v=DMARC1; p=reject; adkim=s; aspf=s`
- Side note: `autosicar.com.mx` (the domain that *does* carry email) has no DMARC record either — worth adding one there too.

## 3. Subdomains — 🟠 Wildcard swallowed everything

Every subdomain tested (`mail`, `webmail`, `cpanel`, `ftp`, `autodiscover`, `blog`, `shop`, `api`, `app`, `admin`, `dev`, `staging`, …) now resolves to Vercel edge IPs. This means:

- Any **service subdomains from the old host** (`mail.`, `webmail.`, `cpanel.`, `ftp.`, `autodiscover.` for Outlook clients) no longer point at real services. Mail clients configured against `mail.sicarautos.com.mx` will fail to connect.
- Subdomains without an assigned Vercel deployment will show Vercel's `DEPLOYMENT_NOT_FOUND` error to visitors instead of NXDOMAIN.

**Action:** inventory which subdomains were actually in use before the migration and re-create only those records in Vercel DNS pointing at the right services.

## 4. TLS / Certificates

- Port 443 accepts TLS 1.3 connections for both apex and `www` — the Vercel edge is serving the domain.
- **CAA records are present and correct** for Vercel-managed certificates: `letsencrypt.org`, `pki.goog` (Google Trust Services), `sectigo.com`. ✅ This is a good security posture — only these CAs can issue certs for the domain.
- ⚠️ The actual leaf certificate could **not** be verified from this environment (the egress gateway re-terminates TLS). Verify externally that Vercel has issued the production cert for both hostnames — e.g. check the padlock in a browser or https://crt.sh/?q=sicarautos.com.mx — especially since certificate issuance is one of the last steps to converge after a DNS move.

## 5. DNSSEC — ℹ️ Not enabled

No DS/DNSKEY records. Vercel DNS does not support DNSSEC, so this is a platform constraint rather than a misconfiguration. Acceptable for most businesses; noted for completeness.

## 6. Not auditable from this environment (follow-up needed)

Blocked by the network egress policy — recommend re-running once the domain is allowlisted, or checking manually:

- HTTP→HTTPS and apex↔`www` redirect behavior (pick one canonical host, 301 the other)
- Page content, broken links, `DEPLOYMENT_NOT_FOUND` errors
- Security headers (HSTS, CSP, X-Content-Type-Options, …)
- On-page SEO (title/meta description, `sitemap.xml`, `robots.txt`, structured data, canonical tags) — important after a migration so Google re-crawls cleanly; verify in Google Search Console that the property still validates
- Performance / Core Web Vitals (run [PageSpeed Insights](https://pagespeed.web.dev/) against the live site)

## 7. Business context (from public sources)

The domain belongs to **Autos Sicar**, a semi-new (seminuevos) car dealership in Monterrey, Nuevo León, Mexico (Colón 3385, Col. Madero), with 20+ years in business, active on [Facebook](https://www.facebook.com/AutoSicar/) and [Instagram](https://www.instagram.com/autosicar/), and listed on [seminuevos.com](https://www.seminuevos.com/dealers-profile/nuevo-leon-monterrey-sicar/6775) and [autocosmos.com.mx](https://www.autocosmos.com.mx/sicarautos). A lead-driven business like this makes the missing-email finding (§2) especially urgent — inbound customer inquiries may be bouncing right now.

---

## Priority checklist

| # | Priority | Action |
|---|---|---|
| 1 | 🔴 Now | Verify whether `@sicarautos.com.mx` mailboxes exist; if so, restore MX records in Vercel DNS |
| 2 | 🟠 High | Publish SPF + DMARC (real values if email is used, null/reject policy if not) |
| 3 | 🟠 High | Re-create any legacy service subdomains still in use (`mail`, `autodiscover`, etc.) |
| 4 | 🟡 Medium | Externally verify the TLS cert issued for apex + `www`, and that redirects canonicalize to one host |
| 5 | 🟡 Medium | Re-validate the site in Google Search Console; confirm `sitemap.xml`/`robots.txt` on the new host |
| 6 | ⚪ Later | Run the blocked content/performance/SEO checks (allowlist the domain for a follow-up audit) |
