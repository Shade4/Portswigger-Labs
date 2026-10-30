# Brute-Forcing 2FA Codes & Weaknesses in "Remember Me" Cookies

Login forms tend to get the most security scrutiny because they're the obvious front door. But a few steps away from that front door sit three mechanisms that are just as capable of handing an attacker a full account takeover: the 2FA verification step, the account-management pages that sit next to login (password change/reset, registration), and the "remember me" cookie that quietly lets a user skip login entirely. This writeup covers all three — and finishes with a PortSwigger lab that puts the "remember me" weaknesses into practice.

---

## 1. Brute-Forcing 2FA Verification Codes

A 2FA code is usually a 4 or 6-digit number. That's only 10,000–1,000,000 possible values — trivial for Burp Intruder to exhaust in minutes if nothing stands in the way. The two most common "protections" sites rely on are:

- **A short code lifetime** (the code expires after 30–120 seconds)
- **Logging the user out after N wrong attempts**, forcing them to re-enter their password before trying again

The second one feels like a real control but isn't. An attacker doesn't need to do anything manually — a Burp macro (or the **Turbo Intruder** extension, which can drive multi-step, stateful attacks far faster than standard Intruder) can replay the full password-then-code sequence automatically: submit the password, try a code, get logged out, log back in with the known password, try the next code, repeat. The "logout" just becomes one more scripted step in the attack loop, and at Turbo Intruder's request rates, even a short-lived code can be brute-forced within its validity window if the rate limit isn't actually enforced server-side.

The real flaw, almost always, is that **rate-limiting is enforced on something the attacker controls** — a cookie, a header, or the client IP — rather than being tied to the account being targeted.

### Real-world example

In November 2025, the pentest firm **Securitum** published a case (dubbed *"One-Time-Pwn"*) where a Next.js/React application's email-OTP login looked solid on paper: a random 6-digit code, three verification attempts per code, and a one-minute cooldown between new codes. In practice, the rate limiter trusted the `X-Forwarded-For` header, and requests that claimed to originate from `localhost` were exempted from any limiting at all. By stripping the `Cookie` and `User-Agent` headers and setting `X-Forwarded-For: localhost`, the testers made both the "send code" and "verify code" endpoints think every request was internal, bypassing the per-IP and per-session limits entirely. Running this through Intruder, they reached a valid OTP and full admin account takeover after roughly 267,000 verification attempts and 120,000 triggered code-send requests — all inside about 90 minutes. The lesson: a rate limit that trusts a client-suppliable header isn't a rate limit, it's a honor system.

A related, simpler variant shows up constantly in CVE databases too: 2FA endpoints with **no rate-limiting or lockout mechanism at all** on the OTP-verification request, so once an attacker has valid credentials, the second factor adds no real protection — it's just extra requests to send.

---

## 2. Vulnerabilities in Other Authentication Mechanisms

Login isn't the only entry point into the authentication system. Password-change pages, password-reset flows, and registration forms all perform authentication-adjacent logic, but they're frequently built and reviewed with far less care than the login form itself — teams harden the door they think of as "the" door and forget the side doors lead to the same house. This matters even more when an attacker can self-register, because a throwaway account gives them a fully authenticated, white-box view of pages they'd otherwise only see from the outside.

One of the most common bugs in this category is **password-reset poisoning**. If the reset-password email is built using the `Host` header from the incoming request instead of a hardcoded, server-side domain, an attacker can request a password reset for a victim's email address while supplying their own domain in the `Host` header. The victim receives a completely legitimate-looking reset email — but the link in it points at the attacker's server, so when the victim clicks it, the attacker captures the reset token and can set a new password on the victim's account without the victim ever realizing anything was wrong.

### Real-world example

A researcher testing HubSpot's public bug bounty program (run on Bugcrowd) found exactly this chain, with HubSpot's defenses falling one layer at a time: the raw `Host` header was validated, so they moved to `X-Forwarded-Host` (used by the app because a load balancer sits in front of the origin, and the app trusted that header instead); when that was also checked, they tried the `Referer` header, then the `Origin` header; eventually they found the server-side domain validation used a loose regex that could be bypassed with a crafted subdomain. Chaining the working bypass into a reset request pointed the victim's reset email at an attacker-controlled domain, giving full account takeover — all through the password-reset flow, without ever touching the login form itself.

---

## 3. Keeping Users Logged In — Weaknesses in Persistent-Login Cookies

### 3.1 Predictable Token Construction

The "Remember me" / "Keep me logged in" checkbox is usually implemented by minting a long-lived token and storing it in a persistent cookie. Possessing that cookie is functionally equivalent to knowing the username and password — it skips the entire login flow — so the token needs to be as unguessable as a strong password. In practice, some implementations instead build the token from predictable, static ingredients:

```
token = username + ":" + account_creation_timestamp
# or, worse:
token = username + ":" + hash(password)
```

If an attacker can register their own account, they can study their own "remember me" cookie, work out the formula, and then generate valid cookies for other usernames — especially if a timestamp or other low-entropy value is part of the recipe and can be brute-forced or guessed from publicly visible account-creation dates. Even without a self-registered account, an attacker who steals one victim's cookie (via XSS, for instance) can reverse-engineer the construction and forge cookies for other users from there.

**Real-world example:** CVE-2023-49259, affecting the Hongdian H8951-4G-ESP cellular router, is a clean textbook case: the device's authentication cookie was generated from the username, a hardcoded secret baked into the firmware, and the device's uptime — all either static or trivially observable — making the cookie guessable within a practical timeframe. It's catalogued under CWE-341 ("Predictable from Observable State") for exactly this reason: nothing about the token depended on an actual secret the attacker couldn't obtain or infer.

### 3.2 The False Security of "Encryption"

Knowing that plaintext concatenation is risky, some developers reach for encoding or hashing and assume the problem is solved. Two common half-measures:

- **Base64-"encrypting" the cookie.** Base64 is an encoding, not encryption — it's trivially reversible with no key required. Wrapping a predictable value in Base64 doesn't hide the formula, it just adds one `atob()` call to the attacker's workflow.
- **Hashing with a well-known algorithm and no salt.** If the attacker can identify the hash function (often obvious from the output length/format), they can hash entire wordlists and known breach-dump password lists offline and compare — and crucially, this bypasses any rate-limit or lockout the site enforces on the *login form*, because the attacker isn't sending requests to the server at all while cracking.

Even properly implemented encryption or HMAC-signing is only as strong as the secrecy of the key. If that key is shared across installs, hardcoded in a public repo, or otherwise exposed, "encrypted" cookies become forgeable by anyone who has the key — no cryptographic weakness required.

**Real-world example:** In February 2025, Microsoft disclosed that it had identified **over 3,000 publicly exposed ASP.NET `machineKey` values** sitting in public GitHub, GitLab, and Bitbucket repositories — often copy-pasted from old tutorials or sample configs and never regenerated. That key is what ASP.NET uses to sign and encrypt `ViewState` data and Forms Authentication cookies: the data is Base64-encoded and protected with a MAC computed from the `machineKey`, so on paper it's "encrypted." But because thousands of sites were using the *same*, publicly known key, attackers could construct and sign malicious payloads themselves and have the server accept them as legitimate — leading to pre-authentication remote code execution on affected IIS servers. The same root issue resurfaced later in 2025: a Sitecore zero-day (CVE-2025-53690) was exploitable because the product's deployment documentation had shipped a sample `machineKey` that customers deployed unchanged into production. The cryptography itself was never broken — the key just wasn't a secret.

### 3.3 Lab Walkthrough: Brute-Forcing a Stay-Logged-In Cookie

**PRACTITIONER**

This PortSwigger lab lets users stay logged in after closing their browser, and the persistent cookie behind that feature turns out to be brute-forceable.

**Goal:** crack Carlos's stay-logged-in cookie and reach his My Account page.

- Own credentials: `wiener:peter`
- Victim: `carlos`

<details>
<summary>Candidate password list used for the attack</summary>

```
123456
password
12345678
qwerty
123456789
12345
1234
111111
1234567
dragon
123123
baseball
abc123
football
monkey
letmein
shadow
master
666666
qwertyuiop
123321
mustang
1234567890
michael
654321
superman
1qaz2wsx
7777777
121212
000000
qazwsx
123qwe
killer
trustno1
jordan
jennifer
zxcvbnm
asdfgh
hunter
buster
soccer
harley
batman
andrew
tigger
sunshine
iloveyou
2000
charlie
robert
thomas
hockey
ranger
daniel
starwars
klaster
112233
george
computer
michelle
jessica
pepper
1111
zxcvbn
555555
11111111
131313
freedom
777777
pass
maggie
159753
aaaaaa
ginger
princess
joshua
cheese
amanda
summer
love
ashley
nicole
chelsea
biteme
matthew
access
yankees
987654321
dallas
austin
thunder
taylor
matrix
mobilemail
mom
monitor
monitoring
montana
moon
moscow
```

</details>

**Step 1 — Generate a reference cookie.**

Log in as `wiener:peter` with the "Stay logged in" box ticked.

![login with wiener](Images/Lab%208/login_with_given_credentials.png)

In Burp's HTTP history, find the login `POST` request and the stay-logged-in cookie it sets in the response.

![stay-logged-in cookie of wiener](Images/Lab%208/request_of_stay_logged_in_wiener.png)

**Step 2 — Reverse-engineer the cookie's format.**

Copy the cookie value into the Decoder tab, and decode it as Base64.

![decoding cookie in base64](Images/Lab%208/stay_logged_in_cookie_wiener_decode.png)

The decoded value comes back as `wiener:<MD5 hash>` — and that hash is just `MD5(peter)`, i.e. MD5 of the account password. So the cookie is built as:

```
cookie = base64( username + ":" + md5(password) )
```

No salt, no server-side secret — just a reversible encoding wrapped around a predictable concatenation. You can reproduce this yourself in Intruder's payload-processing rules: add a "Hash" step (MD5), then an "Add prefix" step (`wiener:`), then a "Base64-encode" step, and feed candidate passwords into the payload list.

**Step 3 — Brute-force the hash with Intruder.**

Run a Sniper attack against the cookie parameter, with the candidate password list as the payload and the three processing rules above applied to each one, so Intruder sends one full candidate cookie per request.

![payload configuration](Images/Lab%208/intruder_configuration_for_session_cookie_brute_force.png)

To spot which attempt actually authenticates as Carlos, add a Grep - Extract rule on something that only shows up for a logged-in account owner on the My Account page — here, the "Update email" button/string.

![update email button](Images/Lab%208/update_email_button.png)
![grep extract](Images/Lab%208/grep_extract_update_email_button.png)

**Step 4 — Identify the winning request.**

Run the attack. Exactly one response has the "Update email" string in the Grep - Extract column — that candidate cookie is the one that successfully impersonates Carlos, solving the lab.

![stay logged in session found](Images/Lab%208/session_cookie_found.png)

**Note:** this lab cracks the hash by brute-forcing with a small, known password list through Intruder. For cracking an MD5 hash against a real wordlist outside a lab, use an offline tool like **hashcat** rather than a public "MD5 decrypt" site — pasting a hash into a third-party site during a real engagement can leak client data to that site's operator.

---

## Defensive Takeaways

| Mechanism | Weak approach | Resilient approach |
|---|---|---|
| 2FA code verification | Rate-limit/lockout keyed to a client-controlled value (IP, cookie, header) | Rate-limit keyed to the account being verified, enforced server-side, independent of any client-suppliable header |
| Account-management pages (reset/change/register) | Assume they inherit the login page's hardening automatically | Review and test them with the same rigor as the login form, since self-registration gives attackers a white-box view |
| "Remember me" tokens | Predictable concatenation of username, password, or timestamp | Cryptographically random, high-entropy token, unrelated to any guessable account attribute |
| Token confidentiality | Base64 "encryption"; unsalted hashing | Proper encryption/HMAC with a per-deployment secret key that is generated, stored, and rotated securely — never committed to source control |
