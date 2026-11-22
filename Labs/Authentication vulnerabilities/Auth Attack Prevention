# Preventing Attacks on Your Own Authentication Mechanisms

A practical guide to building login systems that hold up against real attackers. Each principle has a plain-language definition, what it looks like in practice, and a real-world example.

> Based on the defensive guidance in PortSwigger's Web Security Academy (Authentication vulnerabilities), rewritten in my own words with added examples and code.

## Table of Contents

1. [Take care with user credentials](#1-take-care-with-user-credentials)
2. [Don't count on users for security](#2-dont-count-on-users-for-security)
3. [Prevent username enumeration](#3-prevent-username-enumeration)
4. [Implement robust brute-force protection](#4-implement-robust-brute-force-protection)
5. [Triple-check your verification logic](#5-triple-check-your-verification-logic)
6. [Don't forget supplementary functionality](#6-dont-forget-supplementary-functionality)
7. [Implement proper multi-factor authentication](#7-implement-proper-multi-factor-authentication)
8. [Quick checklist](#quick-checklist)

---

## 1. Take care with user credentials

**In short:** A perfect login system is worthless if you hand valid credentials (or the usernames that go with them) to an attacker without realizing it.

**What to do**

- Never transmit login data over plain HTTP. Use HTTPS everywhere, and don't stop at serving it: **redirect every HTTP request to HTTPS** and send an HSTS header so browsers refuse to downgrade.
- Audit your app for accidental leaks of usernames or email addresses: public profile pages, "forgot password" responses, API responses, error messages, HTML comments.
- Don't put credentials, tokens or reset links in places that get logged or cached (URLs, referrer headers, analytics).

**Real-world example: Firesheep (2010)**
A browser extension called Firesheep made it trivial to hijack other people's logged-in sessions on open Wi-Fi. Many big sites only encrypted the login form itself and then served everything else over plain HTTP, so session cookies flew across the network in the clear. The tool's publicity pushed major sites to move to HTTPS for the entire session. The lesson: encrypting only the login page is not enough.

---

## 2. Don't count on users for security

**In short:** Security that depends on every user making good choices will fail. People take the easiest path, so the system must enforce secure behavior itself.

**What to do**

- Old-style rules ("8 characters, 1 uppercase, 1 symbol") backfire: users just write `Password1!`. Instead, use a **real-time strength checker** such as [`zxcvbn`](https://github.com/dropbox/zxcvbn) (built by Dropbox) and only accept passwords it rates as strong.
- Check new passwords against lists of known-breached passwords.
- Store passwords with a slow, salted hash (Argon2id, bcrypt, scrypt), never plain text or fast hashes like MD5/SHA-1.

```js
// Example: enforce strength server-side, not just in the browser
import zxcvbn from "zxcvbn";

const result = zxcvbn(password, [username, email]); // penalize passwords that contain user info
if (result.score < 3) {
  return res.status(400).json({
    error: "Password too weak",
    hint: result.feedback.suggestions,
  });
}
```

**Real-world example: RockYou breach (2009)**
The RockYou breach exposed roughly 32 million passwords stored in plain text. Analysis of the leak showed that `123456` and similar guesses were among the most common choices. The leaked list is still used today as a standard wordlist in password-cracking tools, proving that users, left alone, pick predictable passwords, and attackers know it.

---

## 3. Prevent username enumeration

**In short:** Username enumeration means an attacker can tell *which accounts exist* by watching how your site behaves. Once they have a list of valid users, brute-forcing becomes far easier. Sometimes the existence of an account is itself sensitive (a health forum, a dating site, an HR portal).

**Where sites leak it**

| Leak channel | Example |
|---|---|
| Error message text | "Invalid username" vs. "Incorrect password" |
| HTTP status code / redirect | `200` for unknown user, `302` for known user |
| Response length or a hidden character difference | One extra space or trailing period in the message |
| Response time | Hashing only happens when the user exists, so valid users respond slower |
| Registration / reset flows | "That email is already registered" |

**What to do**

- Use one **identical, generic message** for every failure, and make sure it is *truly* identical (same text, same HTML, same length).
- Return the **same status code** every time.
- Make **response times indistinguishable**, for example by always running a password-hash comparison, even when the user doesn't exist.
- Apply the same thinking to password reset ("If that email is registered, we've sent a link") and sign-up flows.

```python
# Always do the expensive hash work, whether or not the user exists
DUMMY_HASH = hash_password("a-random-dummy-password")

def login(username, password):
    user = db.find_user(username)
    stored_hash = user.password_hash if user else DUMMY_HASH
    password_ok = verify_password(password, stored_hash)

    if user is None or not password_ok:
        return 401, {"error": "Invalid username or password"}
    return 200, start_session(user)
```

**Real-world example (illustrative scenario)**
A support forum for a sensitive medical condition tells visitors "No account found for that email" on its reset page. An attacker feeds in a leaked email list and instantly learns which people are members, which is a privacy breach on its own, even before any password is guessed. The fix is the generic "if an account exists, we've emailed you" response.

---

## 4. Implement robust brute-force protection

**In short:** Guessing passwords is cheap to automate, so your job is to make it slow, noisy and unprofitable.

**What to do**

- Apply **strict rate limiting per IP address** *and* per account. Be careful with how you read the client IP: if you trust a spoofable header like `X-Forwarded-For` blindly, attackers can rotate fake IPs and dodge the limit. Only trust it from your own reverse proxy.
- Require a **CAPTCHA** after a small number of failed attempts.
- Alert on bursts of failures and on **credential stuffing** (many different accounts, one password each).
- Watch out for lockout side-channels: if only *real* accounts get locked, that is another enumeration leak.
- Accept that this is not a complete cure. The aim is to make attacks tedious enough that attackers move on to softer targets.

```js
// Express + express-rate-limit (sketch)
import rateLimit from "express-rate-limit";

app.set("trust proxy", 1); // only trust your own proxy hop

app.post("/login", rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,                       // 10 attempts / 15 min per IP
  standardHeaders: true,
}), loginHandler);
```

**Real-world example: 23andMe (2023)**
Attackers reportedly logged in to a small fraction of accounts (on the order of 14,000) using username/password pairs reused from other breaches, a classic credential-stuffing attack. Because of the platform's relative-sharing feature, those few accounts exposed data about millions of other users. Rate limiting, bot detection and mandatory MFA are exactly the layers that blunt this kind of attack.

---

## 5. Triple-check your verification logic

**In short:** Authentication bugs are usually not about weak crypto, they are small logic mistakes. A check that can be skipped or tricked is about as good as no check at all.

**Common logic flaws**

- The server trusts a client-controlled value (cookie, hidden field, parameter) to say *who* is being verified.
- A step is skipped if a parameter is missing, so the attacker just removes it.
- A "remember me" or "trusted device" token is predictable or reusable.
- Steps can be accessed out of order (jumping straight to the "logged in" page).
- Different code paths (web, mobile API, legacy endpoint) enforce different rules.

```python
# BAD: the 2FA step trusts a cookie the user controls
user = db.get_user(request.cookies["account"])        # attacker changes this to "victim"
if request.form["code"] == user.current_otp:
    login(user)

# GOOD: the server remembers who passed step 1
pending_user_id = session.get("pending_2fa_user_id")  # set only after a correct password
if pending_user_id is None:
    abort(401)

if not rate_limiter.allow(pending_user_id) or not verify_otp(pending_user_id, request.form["code"]):
    abort(401)

session.pop("pending_2fa_user_id")
login(db.get_user(pending_user_id))
```

**How to audit:** write tests that deliberately try to skip each step, reorder steps, swap identifiers, replay tokens and omit parameters. Review every authentication code path as a whole, not just the main login form.

**Real-world example: 2FA bypass classes**
Security researchers repeatedly report flaws of the "trust the client" type. A common one is a 2FA verification request that includes the account name in a parameter or cookie, letting an attacker with their own valid login complete 2FA *as someone else*. It is a favorite pattern in bug-bounty reports and in the PortSwigger authentication labs, and it is entirely a logic bug, not a cryptographic one.

---

## 6. Don't forget supplementary functionality

**In short:** Attackers go wherever the defenses are weakest. Password reset, password change, "remember me", account recovery, email change and registration are all authentication surfaces and need the same rigor as the login page.

**What to do**

- Reset tokens must be long, random, **single-use**, short-lived and tied to one account.
- Never build reset links from the incoming `Host` header (this enables reset-poisoning).
- Rate-limit and monitor reset attempts and code guesses.
- Require the current password for sensitive changes, and notify users by email when credentials or recovery details change.
- Make sure "change password" can't be abused to brute-force the current password without limits.

**Real-world example: Instagram password-reset flaw (2019)**
A researcher reported that Instagram's account-recovery flow, which relied on a short numeric code, could be brute-forced because the rate limiting could be sidestepped by spreading requests across many IP addresses and sending them concurrently. The flaw was in a *supporting* feature rather than the login page, and the reporter received a bug bounty for it. The takeaway is that a weak recovery path undermines even the strongest login.

---

## 7. Implement proper multi-factor authentication

**In short:** MFA protects you when a password leaks, but only if it adds a genuinely *different* factor and is implemented correctly.

**What to know**

- Multiple factors means something you **know**, something you **have**, something you **are**. Checking the same factor twice is not MFA.
- **Email codes** are effectively single-factor: whoever controls the mailbox gets in, and mailbox access is often protected by the same password.
- **SMS codes** do check a second factor, but are vulnerable to **SIM swapping** and SS7/interception.
- **Best:** a dedicated authenticator app or hardware key that generates or signs the code on a device. Prefer **passkeys / FIDO2** where possible, since they resist phishing.
- Apply all the lessons from section 5 to the MFA step itself: no skippable steps, limit code guesses, expire codes quickly.
- Beware of "MFA fatigue": limit repeated push prompts and prefer number-matching approvals.

**Real-world examples**

- **SIM swapping (2019):** The Twitter account of the company's own CEO was publicly reported to have been taken over after his phone number was hijacked at the carrier level. It illustrates why SMS is the weakest MFA option.
- **MFA fatigue (2022):** In the widely reported Uber breach, an attacker is said to have spammed a contractor with push-notification approvals until one was accepted, then contacted the contractor directly to pressure them. Push-based MFA needs rate limits and number matching to resist this.

---

## Quick checklist

- [ ] HTTPS everywhere; HTTP redirects to HTTPS; HSTS enabled
- [ ] No usernames/emails leaked in profiles, responses or errors
- [ ] Strength checker (e.g. `zxcvbn`) plus a breached-password check
- [ ] Passwords hashed with Argon2id / bcrypt / scrypt
- [ ] Identical error text, status code and timing for all login failures
- [ ] Per-IP and per-account rate limiting; CAPTCHA after repeated failures
- [ ] Verification state lives on the server, never in client-controlled values
- [ ] Every step of login, reset and MFA flows tested for skipping and reordering
- [ ] Reset, change-password and recovery flows hardened like the login page
- [ ] MFA via authenticator app, hardware key or passkey; guesses rate-limited

## Further reading

- [PortSwigger Web Security Academy: Authentication](https://portswigger.net/web-security/authentication)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
