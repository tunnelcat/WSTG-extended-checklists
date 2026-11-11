# WSTG v4.2+ - Pentester Quick Reference

Fast, per-entry checklist: what to look for, commands, tools. Based on OWASP WSTG + the testing checklist. **For authorized testing only.**

Common toolkit: `Burp Suite` / `ZAP` (proxy), `nmap`, `ffuf`/`feroxbuster` (fuzz), `nuclei` (templates), `curl`, `httpx`, `subfinder`/`amass` (recon), `sqlmap`, `jwt_tool`, `testssl.sh`.

Status key - tick one per finding: `Not Started` · `Pass` · `Issues` · `N/A`.

---

## 1. Information Gathering (INFO)

### WSTG-INFO-01 - Search Engine Recon
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** leaked creds, internal paths, config, docs indexed by search engines.
- Google dorks: `site:target.com`, `site:target.com ext:pdf|xls|conf|bak`, `inurl:admin`, `intitle:"index of"`.
```bash
# Also check: Shodan, censys.io, Google cache, archive.org (Wayback)
curl -s "https://web.archive.org/cdx/search/cdx?url=target.com/*&output=text&fl=original&collapse=urlkey"
theHarvester -d target.com -b all
```
- Tools: Google/Bing/DuckDuckGo, Shodan, `theHarvester`, GitHub search for leaked secrets.

#### Notes

### WSTG-INFO-02 - Fingerprint Web Server
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** server type/version → known CVEs.
```bash
curl -sI https://target.com                 # Server, X-Powered-By headers
nmap -sV -p80,443 --script=http-headers target.com
whatweb -a3 https://target.com
```
- Tools: `whatweb`, `httprint`, `nmap`, Wappalyzer. Banners often spoofed - confirm via behavior (header order, error pages).

#### Notes

### WSTG-INFO-03 - Review Webserver Metafiles
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** disallowed paths, sitemaps, security.txt.
```bash
curl -s target.com/robots.txt
curl -s target.com/sitemap.xml
curl -s target.com/.well-known/security.txt
curl -s target.com/humans.txt
```
- Check `<meta>` robots tags in HTML too.

#### Notes

### WSTG-INFO-04 - Attack Surface / Enumeration
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** subdomains, vhosts, DNS, cert SANs, CT logs.
```bash
subfinder -d target.com -all -silent | httpx -silent
amass enum -passive -d target.com
# CT logs
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u
# vhost brute
ffuf -w vhosts.txt -u https://target.com -H "Host: FUZZ.target.com" -fs <baseline>
```
- Tools: `subfinder`, `amass`, `dnsx`, `crt.sh`, `gau`, `waybackurls`.

#### Notes

### WSTG-INFO-05 - Web Page Content Leakage
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** HTML comments, metadata, JS files, source maps, API keys.
```bash
# pull all JS and grep for secrets/endpoints
gau target.com | grep '\.js$' | httpx -silent | while read u; do curl -s "$u"; done > js.txt
grep -riE "api[_-]?key|secret|token|password|https?://" js.txt
# source maps
curl -s target.com/app.js.map
```
- Tools: `LinkFinder`, `SecretFinder`, `JSMiner`, browser DevTools.

#### Notes

### WSTG-INFO-06 - Identify Entry Points
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** all params, headers, methods - build request/response map.
- Spider via Burp/ZAP; log every GET/POST param, cookie, custom header.
- Note hidden form fields, JSON body params, API routes.

#### Notes

### WSTG-INFO-07 - Map Execution Paths
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** main workflows, state transitions, role-gated flows.
- Crawl with authenticated + unauth sessions; diff. Use Burp sitemap + content discovery.

#### Notes

### WSTG-INFO-08 - Fingerprint Framework
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** framework + version → known vulns.
```bash
whatweb https://target.com
# tells: cookie names (PHPSESSID, JSESSIONID, csrftoken=Django), headers, paths (/wp-admin)
nuclei -u https://target.com -t technologies/
```
- Tools: Wappalyzer, `whatweb`, `retire.js` (JS libs).

#### Notes

### WSTG-INFO-09 - Fingerprint Web Application
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** specific app/CMS + version (WordPress, Joomla, etc.).
```bash
wpscan --url https://target.com --enumerate vp,vt,u
nuclei -u https://target.com -t http/cms/
```

#### Notes

### WSTG-INFO-10 - Map Application Architecture
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** WAF, LB, CDN, reverse proxy, microservices, caches.
```bash
wafw00f https://target.com
nmap --script http-waf-detect,http-waf-fingerprint target.com
```
- Note TLS cert differences, header variance, timing → separate backends.

#### Notes

---

## 2. Configuration & Deployment (CONF)

### WSTG-CONF-01 - Network Infra Config
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** default creds, known-vuln services, open mgmt ports, permissive cloud SG/NSG.
```bash
nmap -sV -sC -p- target.com
nuclei -u https://target.com -t http/default-logins/ -t http/misconfiguration/
```

#### Notes

### WSTG-CONF-02 - App Platform Config
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** sample/default files, debug mode, admin consoles, verbose logging.
```bash
ffuf -w common.txt -u https://target.com/FUZZ -mc 200,403
# sample files: /phpinfo.php /test.php /server-status /actuator /actuator/env
curl -s target.com/actuator/env        # Spring Boot leak
```

#### Notes

### WSTG-CONF-03 - File Extension Handling
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** server treats `.php.bak`, `.inc`, `.old` as text (source disclosure).
```bash
ffuf -w files.txt:FILE -w ext.txt:EXT -u https://target.com/FILE.EXT
# ext list: .bak .old .txt .src .inc .zip .tar.gz .swp .save
```

#### Notes

### WSTG-CONF-04 - Backup & Unreferenced Files
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `index.php~`, `.git/`, `.svn/`, `backup.zip`, editor swap files.
```bash
curl -s target.com/.git/HEAD                 # exposed repo
git-dumper https://target.com/.git/ ./loot
ffuf -w backups.txt -u https://target.com/FUZZ
```
- Tools: `git-dumper`, `dvcs-ripper`, `feroxbuster`.

#### Notes

### WSTG-CONF-05 - Admin Interfaces
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** hidden admin panels, CI/CD, k8s/cloud consoles; default/weak creds.
```bash
ffuf -w admin-panels.txt -u https://target.com/FUZZ -mc 200,301,302,401,403
# common: /admin /manager/html (Tomcat) /jenkins /phpmyadmin /.git /console
```

#### Notes

### WSTG-CONF-06 - HTTP Methods
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** PUT/DELETE/TRACE enabled, method-based authz bypass.
```bash
curl -sX OPTIONS https://target.com -i          # Allow header
curl -X PUT https://target.com/test.txt -d "data"
curl -H "X-HTTP-Method-Override: PUT" https://target.com/resource
nmap --script http-methods --script-args http-methods.url-path=/ target.com
```

#### Notes

### WSTG-CONF-07 - HSTS
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** missing/weak `Strict-Transport-Security`, no preload, short max-age.
```bash
curl -sI https://target.com | grep -i strict-transport
# want: max-age>=31536000; includeSubDomains; preload
```

#### Notes

### WSTG-CONF-09 - File Permissions
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** world-readable secrets, loose perms on configs/keys, broad cloud object ACLs.
```bash
# if shell access obtained
find / -perm -o+w -type f 2>/dev/null
```

#### Notes

### WSTG-CONF-10 - Subdomain Takeover
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** CNAME pointing to deprovisioned service (S3, GitHub Pages, Heroku, etc.).
```bash
subfinder -d target.com | dnsx -cname -resp
nuclei -l subs.txt -t http/takeovers/
subjack -w subs.txt -ssl -v
```

#### Notes

### WSTG-CONF-11 - Cloud Storage
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** public buckets, listable objects, weak presigned URLs.
```bash
curl -s https://bucket.s3.amazonaws.com/        # listing?
aws s3 ls s3://bucket --no-sign-request
```
- Tools: `cloud_enum`, `s3scanner`.

#### Notes

### WSTG-CONF-12 - Content Security Policy
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** missing CSP, `unsafe-inline`, `unsafe-eval`, wildcard sources, bypassable allowlist.
```bash
curl -sI https://target.com | grep -i content-security-policy
```
- Evaluate at csp-evaluator.withgoogle.com.

#### Notes

### WSTG-CONF-13 - Path Confusion
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** trailing slash / encoding / case differences bypassing auth, cache, routing.
```bash
curl -s target.com/admin ; curl -s target.com/admin/ ; curl -s target.com/./admin
curl -s target.com/ADMIN ; curl -s "target.com/admin%2f"
```

#### Notes

### WSTG-CONF-14 - Other Security Headers
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** missing `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`.
```bash
curl -sI https://target.com
nuclei -u https://target.com -t http/misconfiguration/http-missing-security-headers.yaml
```

#### Notes

### WSTG-CONF-15 - Feature Flag Bypass
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** client-controllable flags that gate security; backend not re-enforcing.
- Flip flags in requests/cookies/JSON (`"beta":true`, `"isAdmin":false→true`); confirm server re-checks. Test fail-open when flag service down; look for stale flags exposing old code paths.

#### Notes

---

## 3. Identity Management (IDNT)

### WSTG-IDNT-01 - Role Definitions
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** over-privileged roles; attempt role switch via param/cookie.
- Map every role→permission; test cross-role access.

#### Notes

### WSTG-IDNT-02 - User Registration
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** weak identity proofing, no verification, duplicate/role self-escalation at signup.

#### Notes

### WSTG-IDNT-03 - Account Provisioning
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** who can create accounts/of what type; deprovisioning gaps; orphaned accounts.

#### Notes

### WSTG-IDNT-04 - Account Enumeration
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** different responses/timing for valid vs invalid users (login, reset, register).
```bash
# compare messages & response time
curl -s -d "user=admin&pass=x" target.com/login     # "invalid password" vs "user not found"
```

#### Notes

### WSTG-IDNT-05 - Weak Username Policy
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** predictable usernames (email, first.last), enumeration-friendly formats.

#### Notes

---

## 4. Authentication (ATHN)

### WSTG-ATHN-01 - Creds over Encrypted Channel
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** login form posting over HTTP, mixed content.
```bash
curl -sI http://target.com/login       # redirects to https? form action scheme?
```

#### Notes

### WSTG-ATHN-02 - Default Credentials
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** admin/admin, product default creds, predictable new-account passwords.
- Check vendor docs for defaults; try on admin/app/device panels.

#### Notes

### WSTG-ATHN-03 - Weak Lockout
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** no lockout after N failures; or lockout bypass (case, IP rotation); unlock weakness.
- Try ~10+ bad logins; watch for lockout + how it unlocks.

#### Notes

### WSTG-ATHN-04 - Bypassing Auth Schema
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** direct page access, forced browsing, param tampering (`authenticated=1`), SQLi auth bypass.
```bash
curl -s target.com/dashboard -i        # reachable without session?
```

#### Notes

### WSTG-ATHN-05 - Remember-Me / Password
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** persistent cookie holding creds/predictable token; not invalidated on logout.

#### Notes

### WSTG-ATHN-06 - Browser Cache
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** sensitive pages cacheable (back button after logout shows data).
```bash
curl -sI target.com/account | grep -iE "cache-control|pragma"   # want no-store
```

#### Notes

### WSTG-ATHN-07 - Weak Auth / Password Policy
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** short/simple passwords allowed, no breach check, weak aging/reuse rules.

#### Notes

### WSTG-ATHN-08 - Weak Security Questions
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** guessable/OSINT-able answers, limited answer space, no rate limit.

#### Notes

### WSTG-ATHN-09 - Password Change/Reset
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** reset token predictability/reuse/expiry, host-header poisoning in reset link, no current-pw check on change, user-id tamper.
```bash
# host header poisoning of reset link
curl -s target.com/reset -d "email=victim@x.com" -H "Host: attacker.com"
```

#### Notes

### WSTG-ATHN-10 - Weaker Auth in Alt Channel
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** mobile API / legacy endpoint with weaker auth than main site.

#### Notes

### WSTG-ATHN-11 - MFA
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** MFA bypass - skip step via direct nav, null/brute OTP, no rate limit, backup-code weakness, "remember device" flaw, response manipulation (`"mfa":false`).

#### Notes

---

## 5. Authorization (ATHZ)

### WSTG-ATHZ-01 - Directory Traversal / File Include
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** path params that read files; LFI/RFI.
```bash
curl -s "target.com/?file=../../../../etc/passwd"
curl -s "target.com/?file=....//....//etc/passwd"        # filter bypass
# encodings: %2e%2e%2f , ..%252f , php://filter/convert.base64-encode/resource=
```
- Tools: `ffuf` with traversal wordlist, Burp Intruder.

#### Notes

### WSTG-ATHZ-02 - Bypassing Authz Schema
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** vertical (user→admin) & horizontal (user→user) access; unauth access to protected.
- Replay privileged requests with low-priv/no token; compare.

#### Notes

### WSTG-ATHZ-03 - Privilege Escalation
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** role/param tampering (`role=admin`, `group_id`), mass-assign to privilege field.

#### Notes

### WSTG-ATHZ-04 - IDOR
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** object IDs in URL/body you can increment/swap to access others' data.
```bash
curl -s target.com/api/invoice/1001 -H "Cookie: session=..."   # try 1002,1003
```
- Tools: Burp (Autorize / Auth Analyzer extension).

#### Notes

### WSTG-ATHZ-05 - OAuth Weaknesses
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `redirect_uri` validation flaws, missing `state` (CSRF), token leakage, implicit flow, PKCE absent, scope escalation.
```http
GET /authorize?response_type=code&client_id=x&redirect_uri=https://attacker.com HTTP/1.1
```

#### Notes

---

## 6. Session Management (SESS)

### WSTG-SESS-01 - Session Schema
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** weak/low-entropy tokens, predictable, client-modifiable session data.
- Collect many tokens → Burp Sequencer entropy analysis.

#### Notes

### WSTG-SESS-02 - Cookie Attributes
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** missing `Secure`, `HttpOnly`, `SameSite`; overly broad Domain/Path.
```bash
curl -sI target.com/login | grep -i set-cookie
```

#### Notes

### WSTG-SESS-03 - Session Fixation
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** session ID not rotated after login.
- Set a known SID pre-auth → login → check if same SID still valid.

#### Notes

### WSTG-SESS-04 - Exposed Session Variables
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** session token in URL, logs, referer; cacheable.

#### Notes

### WSTG-SESS-05 - CSRF
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** state-changing requests with no anti-CSRF token / SameSite protection.
- Does the request work without token / with another user's token? Tools: Burp (CSRF PoC generator).

#### Notes

### WSTG-SESS-06 - Logout
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** session valid server-side after logout; token reuse.
- Logout → replay old cookie on protected page.

#### Notes

### WSTG-SESS-07 - Session Timeout
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** no idle/absolute timeout; server honors expired token.

#### Notes

### WSTG-SESS-08 - Session Puzzling
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** session var set in one flow misused to authorize another (e.g., reset flow grants auth).

#### Notes

### WSTG-SESS-09 - Session Hijacking
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** token usable from new IP/UA; no binding; transmitted insecurely.

#### Notes

### WSTG-SESS-10 - JSON Web Tokens
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `alg:none`, weak HMAC secret, `RS256→HS256` confusion, no sig check, sensitive claims, no expiry.
```bash
jwt_tool <token> -M at                    # all tests
jwt_tool <token> -X a                     # alg:none
jwt_tool <token> -C -d wordlist.txt       # crack HMAC secret
hashcat -m 16500 jwt.txt wordlist.txt
```

#### Notes

### WSTG-SESS-11 - Concurrent Sessions
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** unlimited simultaneous sessions; no invalidation on password change.

#### Notes

---

## 7. Injection (INJT)

### WSTG-INJT-01 - Reflected XSS
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** input reflected unencoded in response.
```html
<!-- canary then context-check: ?q=cxsstest1234 -> grep in response -->
<script>alert(1)</script>
"><img src=x onerror=alert(1)>
```
- Tools: `dalfox url https://target.com`, Burp, XSS Hunter.

#### Notes

### WSTG-INJT-02 - Stored XSS
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** persisted input (comments, profile, filenames) rendered to other users.
- Inject canary, view as victim/other page; check all render contexts.

#### Notes

### WSTG-INJT-03 - HTTP Verb Tampering
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** auth rules bound to GET/POST only; bypass via HEAD/other verbs.
```bash
curl -X HEAD target.com/admin ; curl -X FOO target.com/admin
```

#### Notes

### WSTG-INJT-04 - HTTP Parameter Pollution
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** duplicate params parsed inconsistently → filter/logic bypass.
```http
GET /?id=1&id=2 HTTP/1.1
```

#### Notes

### WSTG-INJT-05 - SQL Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** errors/boolean/time differences on `'`, `"`, `)` in params.
```sql
-- detect
?id=1'      ?id=1 AND 1=1 / AND 1=2      ?id=1 AND SLEEP(5)
```
```bash
sqlmap -u "https://target.com/?id=1" --batch --risk=2 --level=3
sqlmap -r request.txt --batch --dbs
```

#### Notes

### WSTG-INJT-06 - LDAP Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** LDAP filter input (login/search); probes `*`, `)(`, `*)(uid=*`.

#### Notes

### WSTG-INJT-07 - XML Injection / XXE
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** XML body parsing; test entity expansion / external entity.
```xml
<!DOCTYPE r [<!ENTITY x SYSTEM "file:///etc/passwd">]><r>&x;</r>
<!-- OOB variant: SYSTEM "http://attacker/x" -->
```

#### Notes

### WSTG-INJT-08 - SSI Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `.shtml`; input reflected into server-parsed page.
```html
<!--#echo var="DATE_LOCAL"-->
```

#### Notes

### WSTG-INJT-09 - XPath Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** XML-backed queries; probes `'`, `' or '1'='1`, `count()`.

#### Notes

### WSTG-INJT-10 - IMAP/SMTP Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** CRLF into mail commands via webmail/contact forms → command injection.

#### Notes

### WSTG-INJT-11 - Code Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** input reaching `eval`/interpreter.
- Probes per lang: `;phpinfo();`, `${7*7}`, time-delay payloads.

#### Notes

### WSTG-INJT-12 - Command Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** input into OS command; separators.
```bash
;id   |id   `id`   $(id)   &&id   %0aid        # watch output/timing
;sleep 5   |ping -c5 attacker
```

#### Notes

### WSTG-INJT-13 - Format String
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `%s %x %n` in user fields causing crash/leak (mostly native apps).

#### Notes

### WSTG-INJT-14 - Incubated / Stored
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** stored payload triggered later via separate recall step; set listener.

#### Notes

### WSTG-INJT-15 - HTTP Response Splitting
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** user input into response headers; inject `%0d%0a`.
```http
GET /?param=foo%0d%0aSet-Cookie:%20x=1 HTTP/1.1
```

#### Notes

### WSTG-INJT-16 - Request Smuggling
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** CL.TE / TE.CL / TE.TE / H2.CL desync between front/back ends.
- Tools: Burp **Smuggler** extension, `smuggler.py`. Test timing-based detection first.

#### Notes

### WSTG-INJT-17 - Host Header Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** app trusts `Host` for links/routing/cache/pw-reset.
```bash
curl -s target.com -H "Host: attacker.com"
curl -s target.com -H "X-Forwarded-Host: attacker.com"
```

#### Notes

### WSTG-INJT-18 - SSTI
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** template eval of input.
```text
${7*7}  {{7*7}}  <%= 7*7 %>  #{7*7}      # 49 => vulnerable; then ID engine
```
- Tools: `tplmap -u "https://target.com/?x=*"`.

#### Notes

### WSTG-INJT-19 - SSRF
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** server fetches user-supplied URL (webhooks, import, PDF, image proxy).
```bash
?url=http://169.254.169.254/latest/meta-data/      # cloud metadata
?url=http://127.0.0.1:port/     ?url=file:///etc/passwd
interactsh-client                                   # OOB detection
```

#### Notes

### WSTG-INJT-20 - Mass Assignment
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** binding extra JSON fields not in UI.
```json
{"username":"bob","role":"admin","verified":true}
```

#### Notes

### WSTG-INJT-21 - CSV Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** exported cells starting with `= + - @` executed in Excel/Sheets.
- Inject `=1+1` into fields that land in CSV export; reopen to verify.

#### Notes

### WSTG-INJT-22 - Prototype Pollution
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** merge/clone of user input reaching `__proto__`.
```json
{"__proto__":{"polluted":true}}
```
```text
?a[__proto__][x]=y
```
- Tools: `ppmap`, DOM Invader (Burp).

#### Notes

### WSTG-INJT-23 - Insecure Deserialization
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** serialized blobs in params/cookies/headers (Java `rO0`, PHP `O:`, .NET, pickle).
```bash
ysoserial CommonsCollections1 'command' | base64
phpggc -l         # list PHP gadget chains
```
- Tools: `ysoserial`, `phpggc`, Burp **Java Deserialization Scanner**.

#### Notes

---

## 8. Error Handling (ERRH)

### WSTG-ERRH-01 - Improper Error Handling
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** errors leaking stack/paths/SQL/versions; fail-open logic.
- Force errors: bad types, oversized input, malformed JSON, missing params.

#### Notes

### WSTG-ERRH-02 - Stack Traces
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** full stack traces exposing framework/paths/internal IPs.
- Trigger 500s; inspect body.

#### Notes

---

## 9. Weak Cryptography (CRYP)

### WSTG-CRYP-01 - Weak Transport Layer Security
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** SSLv3/TLS1.0/1.1, weak ciphers, expired/self-signed cert, no PFS.
```bash
testssl.sh https://target.com
sslscan target.com
nmap --script ssl-enum-ciphers -p443 target.com
```

#### Notes

### WSTG-CRYP-02 - Padding Oracle
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** CBC decryption errors distinguishable from padding errors.
```bash
padbuster <url> <sample> <blocksize>
```

#### Notes

### WSTG-CRYP-03 - Sensitive Info via Unencrypted Channels
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** PII/creds/tokens over HTTP, in URL, or plaintext storage.

#### Notes

### WSTG-CRYP-04 - Weak Crypto Primitives
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** MD5/SHA1 for passwords, ECB mode, hardcoded keys/IVs, weak RNG.

#### Notes

---

## 10. Business Logic (BUSL)

### WSTG-BUSL-01 - Data Validation
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** frontend-only checks; bypass via direct request with out-of-range/typed values.

#### Notes

### WSTG-BUSL-02 - Forge Requests
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** hidden/guessable fields, step-skipping with crafted valid data.

#### Notes

### WSTG-BUSL-03 - Integrity Checks
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** tamper insert/update/delete on data you shouldn't control.

#### Notes

### WSTG-BUSL-04 - Process Timing
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** race conditions, TOCTOU, timing-dependent logic.
- Tools: Burp **Turbo Intruder** (single-packet attack) for races.

#### Notes

### WSTG-BUSL-05 - Function Use Limits
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** actions meant once/N-times reusable (coupons, votes, withdrawals).

#### Notes

### WSTG-BUSL-06 - Workflow Circumvention
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** reorder/skip steps (checkout→confirm without payment).

#### Notes

### WSTG-BUSL-07 - Defenses Against Misuse
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** does app detect/respond to aggressive automated input? (observational).

#### Notes

### WSTG-BUSL-08 - Unexpected File Types
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** type restriction bypass (ext, MIME, magic bytes).
- Upload mismatched type (e.g. script content with image MIME), double extension, legacy null-byte tricks.

#### Notes

### WSTG-BUSL-09 - Malicious File Upload
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** webshell/SVG-XSS/EICAR/polyglot accepted & reachable/executed.
- Confirm storage path + execution; check AV/content validation.

#### Notes

### WSTG-BUSL-10 - Payment Functionality
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** price/quantity tamper (negative qty, `price=0.01`), currency, replay, coupon stacking.

#### Notes

---

## 11. Client-Side (CLNT)

### WSTG-CLNT-01 - DOM-Based XSS
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** sources (`location`, `document.URL`, `referrer`) → sinks (`innerHTML`, `eval`, `document.write`).
- Tools: Burp **DOM Invader**, `#payload` fragment tests.

#### Notes

### WSTG-CLNT-02 - JavaScript Execution
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** input into `eval`, `setTimeout(str)`, `Function()`.

#### Notes

### WSTG-CLNT-03 - HTML Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** unencoded HTML reflected into DOM (no script needed).

#### Notes

### WSTG-CLNT-04 - Client-side URL Redirect
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `location = userinput` → open redirect / `javascript:`.
```text
?returnUrl=https://attacker.com    ?next=javascript:alert(1)
```

#### Notes

### WSTG-CLNT-05 - CSS Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** injected CSS exfiltrating data via attribute selectors.

#### Notes

### WSTG-CLNT-06 - Client-side Resource Manipulation
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** input controlling script/iframe/object src.

#### Notes

### WSTG-CLNT-07 - CORS
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** reflected `Origin`, `ACAO: *` with credentials, null origin trusted.
```bash
curl -s target.com/api -H "Origin: https://attacker.com" -I | grep -i access-control
```

#### Notes

### WSTG-CLNT-09 - Clickjacking
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** no `X-Frame-Options`/`frame-ancestors`; page framable.
```bash
curl -sI target.com | grep -iE "x-frame-options|frame-ancestors"
```
```html
<iframe src="https://target.com"></iframe>   <!-- PoC -->
```

#### Notes

### WSTG-CLNT-10 - WebSockets
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** no origin check (CSWSH), unencrypted `ws://`, no authz, injectable messages.
- Test with Burp (WebSocket history/repeater).

#### Notes

### WSTG-CLNT-11 - Web Messaging
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `postMessage` handlers without origin check; unsafe use of `event.data`.
```bash
# grep JS for message handlers
grep -r 'addEventListener("message"' .
```

#### Notes

### WSTG-CLNT-12 - Browser Storage
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** tokens/PII in `localStorage`/`sessionStorage`/IndexedDB; injectable reads.
- DevTools → Application → Storage.

#### Notes

### WSTG-CLNT-13 - Cross Site Script Inclusion (XSSI)
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** sensitive data in JS/JSONP includable cross-origin via `<script>`.

#### Notes

### WSTG-CLNT-14 - Reverse Tabnabbing
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `target=_blank` links / `window.open` without `rel=noopener`.
```bash
grep -rn 'target="_blank"' .    # check for missing noopener
```

#### Notes

### WSTG-CLNT-15 - Client-side Template Injection
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** `{{...}}` eval in Angular/Vue; input into client template.
```text
{{constructor.constructor('alert(1)')()}}      # sandbox escape (AngularJS)
```

#### Notes

---

## 12. API Testing (APIT)

### WSTG-APIT-01 - API Reconnaissance
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** documented + hidden endpoints, params, versions.
```bash
curl -s target.com/swagger.json  /openapi.json  /api-docs  /graphql
ffuf -w api-wordlist.txt -u https://target.com/api/FUZZ    # /api/v1 /api/v2
```
- Tools: `kiterunner`, Postman, mine JS for routes.

#### Notes

### WSTG-APIT-02 - Broken Object Level Authorization (BOLA)
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** object ID in request accessible across users (API IDOR).
- Swap IDs with low-priv token; Burp **Autorize**.

#### Notes

### WSTG-APIT-03 - Excessive Data Exposure
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** API returns more fields than UI shows (hashes, tokens, PII, internal IDs).
- Inspect raw JSON vs rendered page.

#### Notes

### WSTG-APIT-04 - Broken Function Level Authorization (BFLA)
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** low-priv user calls admin functions/methods.
- Replay admin endpoints + swap GET↔POST↔PUT↔DELETE with low-priv token.

#### Notes

### WSTG-APIT-99 - GraphQL
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** introspection enabled, no depth/cost limits, batching abuse, field-level authz gaps.
```bash
curl -s target.com/graphql -d '{"query":"{__schema{types{name}}}"}' -H "Content-Type: application/json"
```
- Tools: `graphql-cop`, `clairvoyance` (schema recovery), InQL (Burp).

#### Notes

---

## 13. WebAssembly (WASM)

### WSTG-WASM-01 - WebAssembly Reconnaissance
- [ ] Not Started
- [ ] Pass
- [ ] Issues
- [ ] N/A
- **Look for:** all `.wasm` modules (incl. renamed/embedded/lazy-loaded), toolchain, debug info (DWARF/name section), embedded URLs/IPs/keys.
```bash
wasm2wat module.wasm -o module.wat
wasm-objdump -x module.wasm
strings module.wasm | grep -iE "http|key|token|/api/"
```
- Tools: `wabt`, `wasm-decompile`, browser DevTools (Sources → wasm).

#### Notes

---

### General Notes
- Always get written authorization & defined scope before testing.
- Confirm findings manually - scanners produce false positives.
- Record requests/responses as evidence; map each finding to its WSTG ID + a risk rating.
