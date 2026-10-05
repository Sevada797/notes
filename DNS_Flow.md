# DNS Flow
Let's write down the basics first

DNS = distributed database of **records**, name --> data.
It only answers questions like "what is the A record of www.domain.com ?", it never connects anything. The client gets the answer and does the actual connecting itself (TCP etc.)


## Who is involved
- **Stub resolver**: tiny client in the OS/app, asks the questions, has its own cache
- **Recursive resolver**: does the whole work for the client, caches answers (ISP, `8.8.8.8`, `1.1.1.1`)
- **Root servers** (`.`) --> **TLD servers** (`.com`) --> **Authoritative servers** (the NS of `domain.com`, they hold the real records)

Transport: UDP `53` by default, TCP `53` for big answers / zone transfers (AXFR). Encrypted variants: DoT `853`, DoH `443`


## Flow: browser opens https://www.domain.com

```
browser --> OS cache / hosts file --> stub --> recursive resolver
                                                  |
                          root (.)  <-------------+   "who knows .com ?"      --> referral to .com TLD
                          .com TLD  <-------------+   "who knows domain.com ?" --> referral to ns1.provider.net
                          authoritative <---------+   "A www.domain.com ?"     --> answer: 203.0.113.10 (TTL 300)
```

1) Browser/OS cache and hosts file (`/etc/hosts`) are checked first
2) Stub asks the recursive resolver: `A www.domain.com ?`
3) If not cached, recursive walks the tree: root --> TLD --> authoritative, each one either refers to the next or answers
4) Answer comes back and is cached for the **TTL**
5) Client now has an IP, and **only now** opens TCP to `IP:port`
   - Port is **not** from DNS, it comes from the scheme/URL: `https` = `443`, `http` = `80`, or explicit `:8080`
   - (Exceptions: SRV and HTTPS/SVCB records can carry port info, rarely used for web. Mail is hardcoded: SMTP server to server always `25`)
6) TCP handshake --> TLS handshake (**SNI** = `www.domain.com`) --> HTTP request with `Host: www.domain.com`

That's why one IP can serve many sites: IP only gets you to the server, SNI/Host header tells it which site you want

If several IPs are returned, the client picks one or tries them in turn (round robin). If AAAA exists too, clients usually try IPv6 first and fall back to IPv4


## Record types

| Type | Meaning |
|------|---------|
| `A` | name --> IPv4 |
| `AAAA` | name --> IPv6 |
| `CNAME` | name --> another name (alias) |
| `MX` | mail servers of the domain (priority + hostname) |
| `TXT` | free text, used for SPF, DKIM, DMARC, verifications |
| `NS` | which servers are authoritative for the zone |
| `SOA` | zone metadata (primary NS, serial, refresh timers) |
| `PTR` | reverse: IP --> name (`x.x.x.x.in-addr.arpa`) |
| `SRV` | service --> host + port |
| `CAA` | which CAs may issue certs for the domain |


## A record
```
www.domain.com.   300   IN   A   203.0.113.10
```
name, TTL (seconds), class, type, value. This IP is what the client uses for the TCP connection


## CNAME
Alias: "this name is just another name for that one". Resolver sees the CNAME, then **restarts the lookup** for the target and follows until it reaches an A/AAAA:

```
www.domain.com.   300   IN   CNAME   domain.cdn.net.
domain.cdn.net.    60   IN   A       203.0.113.10
```

Client asks for `A www.domain.com`, recursive follows the chain, and the final answer contains the whole chain + the IP.

- Used when the target IP is managed by someone else (CDN, SaaS, cloud LB), so they can change IP and you don't need to touch anything
- CNAME --> CNAME chains are allowed, but keep them short (each hop = more lookups, resolvers cap the length, loops break resolution)
- TTL: each record in the chain keeps its own TTL


## What if both A and CNAME exist on the same name ?
**Not allowed.** A CNAME means "this name is an alias, look at the other name for *everything*", so no other record types can sit on the same name (only DNSSEC stuff like RRSIG/NSEC).

What happens in practice if it still gets there:
- Many authoritative servers (e.g. BIND) refuse to load the zone, and DNS provider panels usually block it
- If it slips through, behavior is **undefined**: some resolvers use the CNAME, some the A, and it can differ per resolver or flip depending on cache state --> random "works for me" outages
- Same applies to **any** other type next to a CNAME: MX, TXT, NS... e.g. a CNAME on a name that also has a TXT = the TXT (SPF/DMARC/verification) becomes unreliable

Related rules:
- **CNAME at the zone apex** (`domain.com` itself) is not allowed, as apex must have `SOA` + `NS` (and usually MX, TXT). That's why providers invented `ALIAS` / `ANAME` / "CNAME flattening": not real DNS records, the provider resolves the target itself and serves plain A records
- **MX and NS targets should not be CNAMEs**, they must point to a name that has A/AAAA directly
- It's fine and common for a name that has **only** that CNAME, e.g. `selector1._domainkey.domain.com CNAME ...` (DKIM at an email provider), or `_dmarc.domain.com`


## Missing records: NXDOMAIN vs NODATA
- **NXDOMAIN**: the name doesn't exist at all
- **NODATA** (`NOERROR` with empty answer): the name exists, but not with that record type

This matters for mail: `MX domain.com` returns NODATA --> sender falls back to the A record (see SMTP note). NXDOMAIN --> mail bounces.


## Wildcards
```
*.domain.com.   300   IN   A   203.0.113.9
```
Answers for names that **don't exist** under `domain.com`. If a name exists already (with *any* type), the wildcard doesn't apply to it, e.g. `mail.domain.com` with only a TXT gives NODATA for A, not the wildcard IP.


## No inheritance in DNS itself
Every name is looked up on its own: `sub.domain.com` doesn't get anything from `domain.com`, except via wildcards.

So when "subs fall to the domain's policy" happens (DMARC), it's not DNS doing it, it's the **receiver's logic** doing a second lookup on the parent domain. SPF and DKIM don't do that (see Syntax in the SMTP note).


## TTL & caching
- Every answer is cached for its TTL, at the recursive resolver and often also at OS/browser
- Low TTL = changes spread quickly, more queries. High TTL = fewer queries, slow changes
- Negative answers (NXDOMAIN/NODATA) are cached too, time taken from the SOA
- When testing, remember changes are not instant, and different resolvers may show different answers for a while


## dig cheat sheet
```
dig A www.domain.com +short
dig CNAME www.domain.com
dig MX domain.com
dig TXT _dmarc.domain.com
dig NS domain.com
dig +trace www.domain.com            # walk root --> TLD --> authoritative
dig A www.domain.com @8.8.8.8        # ask a specific resolver
dig AXFR domain.com @ns1.domain.com  # zone transfer attempt
```


## Bug hunter angles
- **Dangling CNAME takeover**: `shop.domain.com CNAME shop-123.somesaas.com`, but the SaaS account/resource was deleted. Attacker registers that name on the SaaS and now serves content on `shop.domain.com` (phishing, cookies scoped to the parent domain, trust of the main domain)
- **Dangling A**: points to a cloud IP that was released, can be re-grabbed
- **NS delegation takeover**: subdomain delegated with NS to a provider/zone that no longer exists, claim the zone = full control of that subtree
- **AXFR enabled**: authoritative server hands out the entire zone, shows all subdomains
- **Dangling SPF include / DKIM CNAME**: same idea, see Syntax section in the SMTP note
- **Missing/weak DMARC**: spoofing finding, usually low severity unless there's a real inbox PoC
