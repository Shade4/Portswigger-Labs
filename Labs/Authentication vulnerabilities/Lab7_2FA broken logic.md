# Flawed Two-Factor Authentication (2FA) Verification Logic

## What It Is

Two-factor authentication is supposed to add a second, independent checkpoint after a user proves they know the password. But that only works if the server can guarantee that the person completing step two is the *same* person who just completed step one. A lot of real-world 2FA implementations quietly break that guarantee.

The pattern looks like this:

1. The user submits `username` + `password` to a first-step endpoint.
2. The server accepts those credentials and stores *which account is mid-login* — usually in a cookie, a hidden form field, or some other client-visible value (e.g. `Set-Cookie: account=carlos`).
3. The user is sent to a second-step endpoint and asked for a verification code (OTP, SMS code, TOTP, backup code, etc.).
4. When the code is submitted, the server looks at that same client-supplied value to decide **whose OTP to check the code against**.

The flaw is step 4: the server is trusting a value the client can freely edit to determine identity, instead of relying on a server-side session that was already tied to the credentials verified in step 1. Change the cookie or field to a different username, and the server will happily check your guessed code against *that* account instead of your own.

## Why It Happens

At its core, this is an **authentication state that isn't bound to the client**. The server correctly verifies `username:password` in step one, but then hands the "which account are we authenticating" decision back to the client for step two, rather than keeping it server-side and tamper-proof. Common causes:

- The "pending login" identifier is stored in a plain, unsigned cookie or hidden field instead of a signed session token.
- The second-step endpoint re-derives the account from a request parameter (which takes precedence) instead of from server-side session state (which should take precedence).
- Developers treat 2FA as a bolt-on check ("did *a* valid code come in?") rather than as a binding check ("did the valid code for *this specific, already-authenticated* session come in?").

## Anatomy of the Attack

Using the account-cookie version of the flaw as a concrete example:

```
POST /login-steps/first HTTP/1.1
Host: vulnerable-website.com

username=carlos&password=qwerty
```

```
HTTP/1.1 200 OK
Set-Cookie: account=carlos
```

```
GET /login-steps/second HTTP/1.1
Cookie: account=carlos
```

An attacker who owns their **own** valid account can log in normally through step one, then simply overwrite the `account` cookie with a victim's username before submitting the verification code:

```
POST /login-steps/second HTTP/1.1
Cookie: account=victim-user

verification-code=123456
```

If the server doesn't check "is `victim-user` the account that actually passed the password check in this session?", it will validate `123456` against the victim's OTP secret instead of the attacker's. Combine that with a brute-forceable, non-rate-limited code (most OTPs are 4–6 digits), and an attacker never needs to know the victim's password at all — just their username, plus enough requests to guess the code.

## Why This Is Dangerous

- It **collapses two factors into one**: the "something you know" (password) check is completely detachable from the "something you have" (code) check, because the server never re-verifies they belong to the same person.
- It turns 2FA from a defense *against* credential-stuffing/leaked-password attacks into something that can be bypassed with **just a username** — arguably worse than having no 2FA, since it creates a false sense of security.
- It's trivial to automate: Burp Intruder (or any scripted client) can brute-force the code field while holding the victim's identifier constant across every request.

## Real-World Example: GitLab's `find_user` Precedence Bug

A well-documented case of this exact class of bug turned up in GitLab Community Edition's login flow. GitLab's `SessionsController` had a `find_user` method used during the OTP-verification step to figure out which account's code was being checked. It could resolve the user in two ways: from a `params[:login]` value sent in the request, or from `session[:otp_user_id]`, a value set server-side after the password step succeeded.

The bug was that the request parameter took precedence over the server-side session value, so if an attacker included a login parameter while submitting an OTP attempt, the code was checked against whichever account that parameter named rather than the account tied to the authenticated session. In other words, exactly the scenario above, just with a request parameter playing the role of the `account` cookie. As a side effect, the bug also let an attacker probe whether an arbitrary username had 2FA enabled at all, since the response differed depending on whether the targeted account required a code.

A related, if less surgical, example is the [2014 PayPal Security Key bypass uncovered by Duo Labs](https://duo.com/labs/research/paypal-2fa-bypass): PayPal's mobile API authentication flow didn't consistently enforce the 2FA flag on an account, so a request crafted to look like it came from a non-2FA session could skip the second factor entirely — again, a case of the *server-side* logic failing to bind "which factors have actually been verified" to the identity making the request.

## How to Prevent It

| Weak approach | Fix |
|---|---|
| Identify the "pending" account from a client-editable cookie/parameter | Store it in a signed, server-side session that the client cannot forge or edit |
| Let a request parameter override session state (GitLab's original bug) | Always resolve identity from server-side session state first; never let client input take precedence |
| Treat step 1 and step 2 as independent checks | Explicitly bind step 2 to the *specific* session/token created after step 1 succeeded, and reject any mismatch |
| Unlimited or slowly-limited code guesses | Rate-limit and lock out verification-code attempts, independent of password-based lockouts |
| Long-lived, reusable "pending 2FA" state | Expire the intermediate state quickly and invalidate it after one use (success or failure) |

The general principle: **nothing about which account is being authenticated should be decided by data the client can edit** once the password step has passed. That decision belongs entirely to server-side session state established at step one.

## Lab Walkthrough: PortSwigger "2FA Broken Logic"

PortSwigger's Web Security Academy has a practitioner lab built around exactly this flaw, which is a good way to see it hands-on. In this lab, the server tracks which account is mid-verification using a `verify` cookie that is completely separate from the `session` cookie — so an attacker who is logged into *their own* session can simply swap the `verify` cookie's value to point the server's OTP check at a different account, without ever leaving their own authenticated session.

**Lab credentials**
- Attacker's own account: `wiener` / `peter`
- Victim account to compromise: `carlos`
- Goal: reach Carlos's account page

**Step 1 — Capture a normal login flow.**
With Burp's proxy running, log in through the lab's `/login` page using `wiener:peter`, then enter the OTP emailed to the `wiener` account to finish logging in. This just populates Burp's HTTP History with the two requests that matter.

**Step 2 — Find the two `/login2` requests in HTTP History.**
There's a `GET /login2` request that carries the account-identifying cookie, and a `POST /login2` request that submits the actual code:

```
GET /login2 HTTP/2
Host: <lab-id>.web-security-academy.net
Cookie: verify=wiener; session=<SESSION_ID>
```

```
POST /login2 HTTP/2
Host: <lab-id>.web-security-academy.net
Cookie: verify=wiener; session=<SESSION_ID>
Content-Type: application/x-www-form-urlencoded
Content-Length: 13

mfa-code=0450
```

Send the `GET /login2` request to **Repeater**, and the `POST /login2` request to **Intruder**.

![GET login request](Images/Lab%207/GET_request_of_login.png)
![POST request of OTP](Images/Lab%207/POST_request_of_OTP.png)

**Step 3 — Repoint the session at Carlos.**
In Repeater, change the `GET` request's cookie from `verify=wiener` to `verify=carlos` and send it. The `session` cookie value stays exactly the same — only `verify` changes. This is the actual vulnerability in action: the server uses `verify` (fully attacker-controlled) rather than anything tied to the authenticated session to decide whose 2FA state is being checked, so this single edit is enough to make the *attacker's own session* start verifying codes against Carlos's account.

**Step 4 — Brute-force Carlos's code in Intruder.**
Switch to the `POST /login2` request already sitting in Intruder. Make the same cookie edit here (`verify=wiener` → `verify=carlos`), then mark the `mfa-code` value as the payload position:

```
mfa-code=0511
```

- Attack type: **Sniper**
- Payload type: **Brute forcer**
- Character set: `0123456789`
- Min length / Max length: `4` / `4`

![Payload configuration](Images/Lab%207/Payload_Configuration.png)

Run the attack. OTP codes are short and this lab has no rate limiting on the verification step, so a full 4-digit keyspace finishes quickly. Once it's done, use the results filter to hide `2xx` responses — a wrong code returns `200 OK` with an error, while the one correct code returns a `302` redirect. That redirect response is your hit.

![OTP captured](Images/Lab%207/OTP_of_carlos.png)

**Step 5 — Replay the flow as Carlos using the brute-forced code.**
Log out of the `wiener` session and return to the login page. Turn on Burp Proxy's intercept, then log in again with `wiener:peter`:

- Forward the initial `POST /login` (credentials) request unchanged.
- When the `GET /login2` request is intercepted, change `verify=wiener` to `verify=carlos` and forward it.
- On the OTP entry page, type any placeholder code. When the corresponding `POST /login2` request is intercepted, change the cookie to `verify=carlos` **and** replace the submitted `mfa-code` with the correct code found in Step 4, then forward it.
- Forward anything else that follows.

The browser lands on Carlos's account page, which solves the lab — all without ever knowing Carlos's password or having access to his inbox.

## Sources

- GitLab CE Issue #14900 — [Bypassing password authentication of users that have 2FA enabled](https://gitlab.com/gitlab-org/gitlab-ce/issues/14900)
- Duo Security / Duo Labs — [Bypass of PayPal's Two-Factor Authentication](https://duo.com/labs/research/paypal-2fa-bypass)
- PortSwigger Web Security Academy — [Multi-factor authentication](https://portswigger.net/web-security/authentication/multi-factor)
