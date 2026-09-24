# Username Enumeration via Response Timing — Detailed Lab Write-Up

| | |
|---|---|
| **Lab** | Username enumeration via response timing |
| **Provider** | PortSwigger Web Security Academy |
| **Category** | Authentication vulnerabilities |
| **Difficulty** | Practitioner |
| **Objective** | Enumerate a valid username, brute-force that user's password, and access their account page |
| **Provided credentials** | `wiener:peter` (a known-valid account used only to observe normal application behavior) |
| **Primary tool** | Burp Suite — Proxy, Intruder (Pitchfork attack) |
| **Core techniques** | Timing side-channel analysis, HTTP status-code analysis, `X-Forwarded-For` header spoofing to defeat IP-based rate limiting |

> **Scope note:** the technique documented here (spoofing client IPs, brute-forcing credentials, timing analysis) is only being demonstrated against PortSwigger's own intentionally vulnerable lab instance, which explicitly authorizes this kind of testing. The same methodology against a system you do not own or have written permission to test would be unauthorized access.

---

## 1. Introduction

This write-up documents the full process of solving the *Username enumeration via response timing* lab, and expands on the original working notes with a deeper explanation of **why** each step works, not just **what** was clicked. The lab combines two distinct weaknesses that are common in real login systems:

1. **A timing side-channel** in the authentication logic, where the server behaves differently — and therefore takes measurably different amounts of time to respond — depending on whether a submitted username exists.
2. **A trust flaw in IP-based brute-force protection**, where the server derives the "client IP" used for rate-limiting from a client-controllable header (`X-Forwarded-For`) instead of the actual TCP connection source.

Neither weakness is unique to this lab. Both map to well-known, named vulnerability classes (CWE-208 *Observable Timing Discrepancy*, and CWE-290 *Authentication Bypass by Spoofing*) that show up regularly in real production software, some examples of which are covered in [Section 7](#7-real-world-occurrences-of-this-vulnerability-class).

---

## 2. Vulnerability Background

### 2.1 What is username enumeration?

Username enumeration is the ability to determine, from the outside, whether a given username corresponds to a real account on a system — without needing to know the password. On its own this seems low-severity, but it is almost always a stepping stone to a more damaging attack:

- It turns a two-unknowns problem (valid username **and** valid password) into a one-unknown problem, dramatically shrinking the search space for credential stuffing or password spraying.
- It confirms which employees, customers, or accounts exist at all, which is valuable for phishing and social engineering.
- Combined with a weak or guessable password policy, it is frequently the first link in a full account-takeover chain — which is exactly what this lab simulates end to end.

Applications usually try to prevent this by returning an identical, generic error ("Invalid username or password") regardless of which part was wrong. The mistake in this lab is that the *response is identical, but the time it takes to produce that response is not.*

### 2.2 The timing side-channel: why response time leaks information

Modern applications store passwords using a slow, deliberately expensive hashing algorithm (bcrypt, scrypt, Argon2, etc.). This is good practice — it makes offline password cracking slower — but it has a side effect: **verifying a password is measurably slower than looking up a username and finding no match.**

A naive (vulnerable) login handler looks conceptually like this:

```python
def login(username, password):
    user = db.find_user(username)

    if user is None:
        # Fast path — no expensive work happens here
        return "Invalid username or password", 200

    # Slow path — only reached if the username exists
    if bcrypt.check(password, user.password_hash):
        return redirect("/my-account")

    return "Invalid username or password", 200
```

Because the expensive `bcrypt.check()` call only executes when `user` is found, the two branches take noticeably different amounts of time:

- **Nonexistent username** → returns almost immediately (a few milliseconds — just a failed database lookup).
- **Valid username** → returns noticeably later, because the server had to run the password-hashing comparison before it could decide the password was wrong.

This lab exploits that gap directly: submit the **same very long, incorrect password** against every candidate username. Every request goes down the slow path if the username is valid, and the extra CPU work needed to process that long input makes the timing gap between "valid username" and "invalid username" large enough to detect reliably over the network, rather than relying on a few milliseconds of bcrypt overhead alone.

### 2.3 Why the lab also has IP-based brute-force protection

A login endpoint that can be hammered with unlimited attempts is trivially brute-forceable regardless of any other defense, so this lab (realistically) also implements a lockout: after a number of failed attempts from the same IP, that IP is temporarily blocked. This is a genuinely useful control — *if* the server can be trusted to know the real client IP.

The flaw is that the application decides the "client IP" from the client-supplied `X-Forwarded-For` request header rather than the actual TCP connection. `X-Forwarded-For` exists so that a trusted reverse proxy or load balancer can tell the backend the original client's IP after forwarding a request — but if the backend blindly trusts this header from *any* source, an attacker can set it to an arbitrary value on every request and appear to originate from a fresh, never-before-seen IP each time, resetting their attempt budget indefinitely. Burp Intruder's **Pitchfork** attack type is what makes it practical to rotate this header value in lock-step with each brute-force guess.

---

## 3. Lab Environment & Prerequisites

- Burp Suite (Community or Professional) configured as the browser's proxy.
- A Chromium-based or Firefox browser routed through Burp's proxy listener, with Burp's CA certificate trusted (standard PortSwigger lab setup).
- The lab instance URL, of the form `https://<lab-id>.web-security-academy.net`.
- Your own throwaway credentials, `wiener:peter`, used only to see what a *normal* login attempt looks like — not part of the exploit path.
- Two word lists supplied by the lab:
  - **101 candidate usernames** (e.g. `carlos`, `root`, `admin`, `test`, `guest` … `autodiscover`).
  - **100 candidate passwords** (e.g. `123456`, `password`, `qwerty` … `moscow`).

---

## 4. Attack Chain Overview

At a high level, the exploitation path has four phases:

1. **Baseline** — capture a normal login POST request to use as an Intruder template.
2. **Enumerate the username** — replay the request once per candidate username, holding a long fixed password constant, and look for the one response that took significantly longer than the rest.
3. **Brute-force the password** — replay the request once per candidate password against the now-known valid username, and look for the one response with an HTTP `302` (redirect) instead of `200` (re-rendered login page).
4. **Bypass the lockout and log in** — because phases 2 and 3 will have already triggered the IP-based lockout on every spoofed IP used so far, submit the final, correct credentials with a *fresh* `X-Forwarded-For` value to reset the attempt counter for that request.

Throughout phases 2–4, the `X-Forwarded-For` header is driven as an Intruder payload position so that every brute-force request appears to come from a different source IP, keeping the request volume under the lockout's per-IP threshold.

---

## 5. Step-by-Step Walkthrough

### Step 1 — Capture a Baseline Request

With Burp's Proxy **Intercept** switched on, open the lab's login page and submit an intentionally wrong credential pair (e.g. `test` / `test`). This isn't meant to succeed — its only purpose is to generate a real `POST /login` request that Intruder can be built from, and to see what a normal "failure" response and its rough response time look like before any timing analysis is attempted.

![POST request of login](Images/Lab%203/POST_request_login.png)
*The intercepted baseline login request — this is the template every subsequent Intruder attack is built from.*

A typical baseline request/response pair looks like this:

```
POST /login HTTP/2
Host: <lab-id>.web-security-academy.net
Cookie: session=<unauthenticated-session-value>
Content-Length: 29
Content-Type: application/x-www-form-urlencoded

username=test&password=test
```

```
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8

... "Invalid username or password." ...
```

Note the response is a plain `200 OK` re-rendering the login form with a generic error — by design, this response gives no information in its *content*. The information leak this lab exploits is entirely in the response **timing**, not the response body.

Send this request to Intruder (right-click → **Send to Intruder**) once it's been observed in the Proxy's **HTTP history** tab.

### Step 2 — Configure the Pitchfork Attack for Username Enumeration

Burp Intruder offers four attack types. Understanding why **Pitchfork** is the correct choice here (rather than the others) matters as much as knowing which button to click:

| Attack type | Behaviour | Why it doesn't fit here |
|---|---|---|
| Sniper | One payload set, cycled through a single position at a time | Only one position can vary — can't drive both the spoofed IP and the username together |
| Battering ram | One payload set, the *same* value inserted into every position simultaneously | Would put the same value into both the IP and username fields, which is meaningless here |
| Cluster bomb | Multiple payload sets, every combination of values tried | Would produce 101 × 101 (or 101 × 100) requests instead of 101 — wasteful and would desynchronize the IP from the username it's meant to accompany |
| **Pitchfork** | Multiple independent payload sets, advanced **in lock-step** by index | Exactly what's needed: request *n* uses IP-list entry *n* together with username-list entry *n* |

In the request template, mark two payload positions:

- **Position 1** — the `X-Forwarded-For` header value (added as a new header if not already present).
- **Position 2** — the `username` parameter.

The `password` parameter is set to a fixed, very long string (not marked as a payload) — commonly several thousand repeated characters (e.g. a long run of `a` characters). This static long password is what amplifies the bcrypt-comparison timing gap enough to be reliably distinguishable from network jitter.

![payload on IP and username](Images/Lab%203/adding_payload_IP_and_username.png)
*Payload markers placed on the `X-Forwarded-For` header and the `username` field, with a long static password left unmarked in the body.*

### Step 3 — Set the Payload Lists

- **Payload set 1** (`X-Forwarded-For`) → payload type **Numbers**, sequential range **1 to 101** (one unique spoofed IP per candidate username, so no single fake IP absorbs more than one failed attempt).
- **Payload set 2** (`username`) → payload type **Simple list**, populated with the 101 candidate usernames supplied by the lab.

![payload configuration first position](Images/Lab%203/payload_position_1.png)
*Numeric range configured for payload position 1 (`X-Forwarded-For`).*

![payload configuration second position](Images/Lab%203/payload_configuration_2.png)
*The candidate username list loaded into payload position 2.*

### Step 4 — Run the Attack and Analyze Response Times

Start the attack. Once it completes, sort the results grid by the **response-time** (or "Response received") column rather than status code or length — every response will return the same generic `200` failure page, so length and status are uninformative here. Look for the single outlier row whose time is dramatically higher than the rest of the pack.

![delay in response](Images/Lab%203/large_delay_in_response_username.png)
*One request stands out with a response time far above the baseline — the tell-tale sign that the server took the "slow path" and actually attempted to verify a password against a real account.*

The pattern typically looks like this (illustrative values — actual timings will vary by network and load):

| Username tested | Approx. response time | Interpretation |
|---|---|---|
| root | ~90 ms | Fast — username doesn't exist |
| admin | ~95 ms | Fast — username doesn't exist |
| test | ~88 ms | Fast — username doesn't exist |
| **carlos** | **~1,100 ms** | **Slow — server hashed/compared the long password, so this username exists** |
| guest | ~91 ms | Fast — username doesn't exist |

The consistently fast responses cluster tightly together (they never reach the password-hashing code path at all), while the one genuine username stands well apart from that cluster. That outlier is the enumerated, valid username.

### Step 5 — Brute-Force the Password by Status Code

Rebuild the attack for the second phase:

- **Username** field is now the fixed, known-valid value from Step 4 (no longer a payload).
- **Password** field becomes payload position 2, loaded with the lab's **Simple list** of 100 candidate passwords.
- **Position 1** (`X-Forwarded-For`) is kept as a **Numbers** payload, but shifted to a fresh, unused range — e.g. **200 to 300** — so this phase doesn't reuse (and re-trip) any IP already spent enumerating usernames in Step 4.

![changing IP range](Images/Lab%203/changing_IP_range.png)
*The `X-Forwarded-For` numeric range moved to 200–300 for the password-guessing phase, avoiding IPs already used (and possibly rate-limited) in the previous attack.*

![password payload position](Images/Lab%203/payload_position_on_password.png)
*Payload position 2 moved onto the `password` field, with the username now static.*

This time, sort the Intruder results by **HTTP status code** rather than timing. Every incorrect guess re-renders the login form (`200 OK`); a correct guess triggers the server to authenticate the session and issue a redirect:

```
HTTP/2 302 Found
Location: /my-account
Set-Cookie: session=<authenticated-session-value>; Secure; HttpOnly
```

![finding password](Images/Lab%203/password_enumeration_change_in_status_code.png)
*The single row returning `302` instead of `200` — this is the correct password for the enumerated account.*

### Step 6 — Bypass the Lockout to Perform the Real Login

At this point the correct username and password are both known — but logging in normally through the browser will likely fail with a message such as *"You have made too many incorrect login attempts. Please try again in 30 minute(s)"*. This is expected: the brute-force phases deliberately generated a large number of failed attempts, and every `X-Forwarded-For` value used in ranges 1–101 and 200–300 is now potentially rate-limited.

The fix follows directly from the same trust flaw already exploited: turn Proxy **Intercept** back on, submit the real login form with the correct username and password, and before forwarding the intercepted request, add (or edit) the `X-Forwarded-For` header to a value that has not been used yet — e.g. `400`, or any number outside both ranges already consumed. From the server's perspective, this is a "new" client with a clean attempt history, so the correct credentials are accepted on the first try.

![bypassing IP block](Images/Lab%203/bypassing_IP_block.png)
*Intercepted final login request, with `X-Forwarded-For: 400` — a value outside both previously used ranges — added before forwarding.*

### Step 7 — Confirm Access

Forward the modified request. The server responds with a `302` redirect to `/my-account` and a valid, authenticated session cookie. Following that redirect in the browser lands on the target account's page, confirming full account takeover and solving the lab.

---

## 6. Summary: The Complete Request/Response Cycle

| Phase | Varying field(s) | Fixed field(s) | What to inspect | Signal that confirms success |
|---|---|---|---|---|
| Baseline | — | — | Response structure, rough timing | Generic 200 "Invalid username or password" |
| Username enumeration | `X-Forwarded-For` (1–101), `username` (list) | long static `password` | **Response time** | One dramatically slower response |
| Password brute-force | `X-Forwarded-For` (200–300), `password` (list) | known `username` | **HTTP status code** | A single `302` among many `200`s |
| Lockout bypass | `X-Forwarded-For` (fresh value) | correct `username` + `password` | Redirect + session cookie | `302 Found`, `Location: /my-account` |

---

## 7. Real-World Occurrences of This Vulnerability Class

This isn't just a contrived lab scenario — the same root cause (an expensive operation, usually password hashing, that only runs on one branch of an authentication check) has been found and fixed in real, widely deployed software on multiple occasions, and is formally tracked as **CWE-208: Observable Timing Discrepancy**.

- **Zabbix web interface — CVE-2024-36469.** Security researcher Jens Just Iversen reported, through a HackerOne bug bounty program, that Zabbix's login page took a measurably different amount of time to reject a login attempt depending on whether the submitted username existed, allowing remote, unauthenticated username enumeration — functionally the same flaw exploited in this lab, found in a monitoring platform used across thousands of production networks.
- **OpenSSH — CVE-2018-15473.** All OpenSSH versions through 7.7 leaked, via a timing/behavioral difference in how malformed authentication packets were processed, whether a supplied username corresponded to a real system account, before any password was even checked. It was fixed in OpenSSH 7.8 (released August 2018), which both corrected the parsing flaw and added further timing-attack countermeasures to the authentication path. This shows the same class of bug isn't limited to HTTP login forms — it applies to any authentication protocol where "does this identity exist" and "is this credential valid" are checked with different amounts of work.
- **Recurring pattern in modern web frameworks.** Multiple, independent disclosures in different products over the following years describe essentially the same mechanism: a database lookup returns immediately for a nonexistent user, while an existing user's request proceeds to a bcrypt (or similar) comparison that measurably delays the response — in some documented cases the gap has been on the order of a few milliseconds for invalid users versus several tens of milliseconds for valid ones. Fixes for these issues typically follow the same pattern: perform a "dummy" password verification even when the username doesn't exist, so every request pays the same computational cost regardless of outcome.

The consistent lesson across all of these real cases is the one this lab is designed to teach hands-on: **any measurable difference between the "valid identity" and "invalid identity" code paths — whether in response content, response time, or anything else — is exploitable**, even when the visible error message is identical in both cases.

---

## 8. Remediation Recommendations

For anyone building or reviewing an authentication system, the defenses that address both halves of this lab are well established:

- **Equalize timing across outcomes.** Always perform a password comparison (even against a dummy hash) regardless of whether the username exists, so the fast and slow paths take statistically indistinguishable amounts of time. Some frameworks implement this as a "timing-safe" authenticator that runs a constant-time dummy verification for unknown users.
- **Use identical, generic responses.** Same error message, same status code, same response size, for both "wrong username" and "wrong password" — this lab shows that identical *content* alone is not sufficient without also equalizing timing, but it remains a necessary baseline control.
- **Never trust client-supplied IP headers for security decisions.** `X-Forwarded-For` (and similar headers) should only be trusted when the request genuinely arrives through a known, controlled reverse proxy that overwrites the header itself; rate-limiting and lockout logic should key off the actual, verified connection source, not a value the client can set arbitrarily.
- **Rate-limit and lock out based on the account being targeted, in addition to (or instead of) source IP**, since IP-based limits alone are inherently spoofable and also generate false positives for legitimate users behind shared/corporate NAT.
- **Add friction against automation generally** — CAPTCHAs after a small number of failures, monitoring/alerting on abnormal login-attempt volume, and, where appropriate, multi-factor authentication so a leaked or brute-forced password alone isn't sufficient for account takeover.

---

## 9. Key Takeaways

- A response that looks identical can still leak information through **how long it took to generate** — timing is a side channel just as much as response content or length.
- Burp Intruder's **Pitchfork** attack type is the right tool whenever two or more payload positions need to advance together in lock-step, rather than being tried in every combination (Cluster bomb) or fixed to identical values (Battering ram).
- Rate-limiting that trusts a client-controllable header (`X-Forwarded-For`) provides little real protection against a determined attacker, since the "identity" it limits on can be freely reset on every single request.
- This exact vulnerability pattern — cheap lookup for invalid identities, expensive verification for valid ones — has recurred in real, production authentication systems (Zabbix, OpenSSH, and others), which is why it's treated as its own named weakness class (CWE-208) rather than a one-off bug.

---

## References

- CWE-208: Observable Timing Discrepancy — https://cwe.mitre.org/data/definitions/208.html
- OWASP Web Security Testing Guide — Testing for Account Enumeration — https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/03-Identity_Management_Testing/04-Testing_for_Account_Enumeration_and_Guessable_User_Account
- CVE-2024-36469 (Zabbix web interface, timing-based username enumeration) — https://db.gcve.eu/vuln/cve-2024-36469
- CVE-2018-15473 (OpenSSH username enumeration) — https://threatprotect.qualys.com/tag/cve-2018-15473/
- PortSwigger Web Security Academy — Authentication vulnerabilities — https://portswigger.net/web-security/authentication
