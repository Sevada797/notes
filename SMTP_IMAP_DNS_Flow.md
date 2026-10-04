# SMTP IMAP DNS Flow
Let's write down the basics first

## Protocols & servers
**SMTP** (ports `25` server to server, `587` submission w/ STARTTLS, `465` implicit TLS): protocol to send emails, also used by receiving end to receive.
There can be several SMTP hops, or none in between client & server

**IMAP** (ports `143` plain/STARTTLS, `993` implicit TLS): protocol to access stored emails, read & manage (e.g. delete)


## Basic structure

```
SMTP packet -->  SMTP HOP  --> SMTP server (receiving end)
```

What does an SMTP HOP do?
It has to check:
1) Is the sender IP allowed by the domain it tries to send from?
2) Who to send/forward to?

How it happens:
1) Take the domain on the right part of "@" (the host) and look up its SPF record (TXT). Compare the sender IP to the allowed IPs/rules in it.
   - Note: SPF is checked against the envelope sender (`MAIL FROM` / `Return-Path`), not the `From:` header. The `From:` header is what the user sees, and that's where DMARC comes in (see below)
2) Send to the SMTP server mentioned in the MX record --> if no MX record, fallback is the main domain's A record (IP)


## How domain verification happens

By using either SPF record, or DKIM (based on cryptography) record

All of them (SPF, DKIM, DMARC) are just **TXT records** in DNS, only the name they live on differs. Syntax details at the end ([Syntax](#syntax))

- **SPF**: lists the IPs (and includes) allowed to send for the domain. TXT record on `domain.com`
- **DKIM**: pub key to use for verifying the signed mail (packet). The sending server signs headers + body with the private key, receiver gets the pub key from DNS: TXT record on `<selector>._domainkey.domain.com`, selector is in the `DKIM-Signature` header (`s=`)

### And DMARC?
DMARC is for configuring what to do with all of this. SPF & DKIM on their own only say pass/fail, they don't tell the receiver what to do about a fail, DMARC does.

TXT record on `_dmarc.domain.com`

What it adds:
1) **Alignment**: SPF or DKIM must pass AND the domain they verified must match the domain in the `From:` header (the one the user sees). Only one of the two needs to pass + align
2) **Policy** (`p=`) for mail that fails:
   - `none`: do nothing, just monitor (report only)
   - `quarantine`: put in spam
   - `reject`: reject at SMTP level, don't accept at all
3) **Reports**: `rua=` aggregate reports, `ruf=` failure/forensic reports (sent to the given mailto)

e.g.
```
_dmarc.domain.com  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@domain.com"
```
this one = reject everything that fails SPF/DKIM alignment, and send daily reports to that mailbox

Other tags: `sp=` (policy for subdomains), `pct=` (% of mail the policy applies to), `adkim=` / `aspf=` (strict `s` or relaxed `r` alignment)

### What if DMARC is not set?
There is **no** fallback of rejecting. No DMARC = the domain owner expressed no wish, so the receiving server decides on its own (local policy):
- SPF/DKIM fail or even both missing --> mail is usually still **accepted**, maybe with a spam score bump or landing in spam folder, depends on the receiver
- Some receivers may still reject on SPF `-all` (hard fail), but that is their choice, nothing forces them. With DMARC present, its policy is what counts instead
- Since there's no alignment check, the visible `From:` header can be anything --> **spoofing the domain is much easier** (SPF/DKIM may pass for the attacker's own domain in `MAIL FROM`, while `From:` shows the victim's)
- Same applies to `p=none`: only monitoring, nothing is blocked

So for a bug hunter: missing DMARC (or `p=none`) on a domain = "email spoofing possible" finding, though many programs consider it low severity / out of scope unless there's a real impact e.g. POC landing not in spam for gmail


## Left part verification ?
This is something that can't be verified (SPF/DKIM/DMARC only cover the domain part)

And that's why mail providers, e.g. gmail, use the SMTP Auth layer (login required on port `587`/`465`)

So that a@gmail.com can't send mail as b@gmail.com, as these are personal mails.


## Syntax
All are TXT records, `tag=value` pairs separated by `;` (SPF is the exception, it's space separated)

| Record | Lives on | Starts with |
|--------|----------|-------------|
| SPF | `domain.com` | `v=spf1` |
| DKIM | `<selector>._domainkey.domain.com` | `v=DKIM1` |
| DMARC | `_dmarc.domain.com` | `v=DMARC1` |

### SPF
Mechanisms are evaluated left to right, first match wins, `all` goes last.

```
domain.com  TXT  "v=spf1 ip4:203.0.113.5 ip4:198.51.100.0/24 include:_spf.google.com mx -all"
```

- `ip4:` / `ip6:`: allowed IP or range
- `a` / `mx`: allow the IPs of the domain's own A / MX records
- `include:`: also accept whatever that domain's SPF allows (used for 3rd party senders)
- `all`: matches everything, so it's the final catch-all, and its qualifier decides the result:

| Qualifier | Meaning |
|-----------|---------|
| `-all` | fail (not allowed) |
| `~all` | softfail (suspicious, usually spam folder) |
| `?all` | neutral (no opinion) |
| `+all` | pass everyone |

Bad cases:
- `+all` or `?all`: SPF is basically useless
- **Two SPF records** on the same name --> permerror, SPF is treated as broken. Must be merged into one
- **More than 10 DNS lookups** (`include`, `a`, `mx`, `redirect`, `exists`, `ptr`; `ip4`/`ip6`/`all` don't count) --> permerror. Happens quickly with many `include:`s
- `include:` pointing to a domain with no SPF record --> permerror
- **Dangling include**: the included domain expired / is unregistered --> someone can register it and control what's allowed for the victim domain (nice for bug bounty)
- `ptr` mechanism: deprecated, slow, avoid
- Over 255 chars in a single TXT string --> must be split into several quoted strings `"...." "...."`, which get joined together by the receiver

Domains/subs that never send mail should have the null record:
```
domain.com  TXT  "v=spf1 -all"
```

### DKIM
```
selector1._domainkey.domain.com  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBgkqh...base64 pub key..."
```

- `v=DKIM1`: version
- `k=`: key type (`rsa` default, or `ed25519`)
- `p=`: the public key (base64)
- `t=y`: testing mode, `t=s`: no subdomains allowed in `d=`
- Selector name comes from the mail itself: `DKIM-Signature: ... d=domain.com; s=selector1; ...`

Bad cases:
- **Empty `p=`** = key revoked, all mails signed with that selector fail
- Weak key (1024 bit or less), use 2048
- 2048 bit keys exceed 255 chars --> split into multiple quoted strings, same as SPF
- `t=y` left on forever: receivers may treat failures as if unsigned
- Often the selector is a **CNAME** to the email provider's key (e.g. Microsoft 365, SendGrid), if that target is dangling/removed --> breaks, or is takeover-able

### DMARC
```
_dmarc.domain.com  TXT  "v=DMARC1; p=reject; sp=reject; rua=mailto:dmarc@domain.com; adkim=r; aspf=r; pct=100"
```

- `v=DMARC1`: must be first tag
- `p=`: policy `none` / `quarantine` / `reject`
- `sp=`: policy for subdomains (if missing, `p=` is used)
- `rua=` / `ruf=`: report addresses, must be `mailto:` URIs
- `adkim=` / `aspf=`: alignment, `r` relaxed (default, subdomains of the same domain match) or `s` strict (exact match)
- `pct=`: % of failing mail the policy is applied to (default 100)

Bad cases (beyond `p=none`):
- **Two DMARC records** or **`v=DMARC1` not first** or typos in tags --> record is ignored, same as having no DMARC
- `pct` below 100 --> only that % of failing mail gets the policy, the rest is treated as one level weaker
- `sp=none` with `p=reject` --> main domain protected, but **subdomains spoofable**
- `rua=` pointing to a different domain without the authorization record (`domain.com._report._dmarc.otherdomain.com TXT "v=DMARC1"`) --> reports silently not sent
- Setting `p=reject` before all legit senders (newsletters, CRM, etc.) have SPF/DKIM aligned --> your own mail gets rejected
- DMARC without SPF **and** DKIM set up properly is pointless, as one aligned pass is needed

### Subdomains
Depends on the record:

- **DMARC**: yes, falls back. If `_dmarc.sub.domain.com` doesn't exist, the receiver looks at the parent (organizational) domain `_dmarc.domain.com` and applies its `sp=` (or `p=` if no `sp=`)
- **SPF**: **no** inheritance. Checked per exact hostname, `sub.domain.com` without its own SPF = result `none`, not the parent's rule
- **DKIM**: **no** inheritance, selector is looked up on the exact domain in `d=` (`s._domainkey.d`)
- **MX**: no inheritance either, but missing MX falls back to that sub's own A record (see Basic structure)

So a sub with no SPF but covered by parent's `p=reject` DMARC: the mail can't align via SPF, so it only passes DMARC if DKIM aligns, otherwise rejected.

### No SPF / no DKIM = not a free pass
With no SPF and no DKIM, the sub publishes no rule about who may send for it and no key to verify anything. Nothing in the sub's own DNS can say "this server is wrong".

But DMARC doesn't ask "was the mail proven fake?", it asks **"can this mail prove it's real?"**, meaning at least one **aligned pass** is required. No SPF or DKIM = no aligned pass possible --> DMARC fails --> policy applies.

- Missing SPF = result `none`, which is **not** a pass
- Missing DKIM (no signature / no key) can't pass either
- Attacker's own SPF/DKIM can pass, but for *their* domain, so it doesn't align with the spoofed `From:`

So under a parent `p=reject` (and no weaker `sp=`) such a sub can't send mail at all, which is ideal for subs that never send.

Only when the policy is weak (`p=none`, `sp=none`, or no DMARC) a fail has no consequence, and then yes, any server can send as that sub.
