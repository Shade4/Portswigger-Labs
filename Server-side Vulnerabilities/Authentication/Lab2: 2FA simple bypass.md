# Bypassing Two-Factor Authentication (Forced Browsing)

This doc covers 2FA bypass in two parts, same pattern as the rest of this repo:

- **Part 1** — what the vulnerability actually is, why it happens, and a real-world example.
- **Part 2** — a hands-on PortSwigger lab where this exact flaw is exploited end-to-end.

---

# Part 1 — The Concept

## Definition — In My Own Words

Two-factor authentication (2FA) is supposed to add a second checkpoint after your password: even if someone steals your password, they still need a code from your phone, email, or authenticator app to actually get in.

The problem is that **many apps build 2FA as a second *page*, not a second *permission check*.** Here's the flaw, step by step:

1. You submit your password.
2. The server validates it and marks your session as "logged in" — a cookie or token is issued right then, before you've touched the verification code field.
3. The front end redirects you to a "Enter your 6-digit code" page.
4. That redirect is just a *suggestion* the app is making to your browser — it's not the same as the server actually blocking access to everything else until the code is confirmed.

If the server only checks "is this session authenticated?" on protected pages — instead of "is this session authenticated **and** has it completed the second factor?" — then you can skip straight past the code screen by simply requesting a protected URL directly. The server never verifies you finished step 2, because it never enforced that check to begin with.

This is really an **authorization gap disguised as an authentication feature**. 2FA is meant to strengthen "prove who you are," but the actual bug lives one layer down — in whether the server checks that the *second* proof was completed before handing out access.

## Why It Happens

- Developers implement 2FA as "step 2 of the login *flow*," relying on the redirect after step 1 to *send* the user to the code page — and forget to also gate the backend session state behind it.
- The session is flagged `authenticated = true` the instant the password check passes. Protected pages then check only for that flag, not a separate `mfa_verified = true` flag.
- As a result, forcibly browsing to an internal URL — typing it directly, following an old bookmark, or replaying a captured request — can succeed, completely skipping the code prompt.

## Real-World Example

Say you log into your bank's website. You enter your username and password, they're accepted, and you land on a screen that says: *"Enter the 6-digit code sent to your phone."*

Behind the scenes, the moment your password was accepted, your session cookie was already stamped `authenticated=true`. You haven't entered the code yet — but that flag doesn't know that.

If you copy your bank's dashboard URL (`https://mybank.com/dashboard`) and paste it into a new tab while you're still sitting on the "enter code" screen, a properly built system sends you straight back to that screen. But if the dashboard page only checks `authenticated=true` and never checks whether MFA was actually completed, it loads anyway — full account balance, transaction history, everything — without you ever having typed the code.

Now flip this into an attacker's shoes: someone has already phished or reused a leaked copy of your password. They don't need to intercept your phone's SMS code at all. They log in with the stolen password, and the instant the app tries to show them the "enter code" screen, they instead type the dashboard URL directly into the address bar — and they're in. The second factor was never actually tested against them.

## How This Differs From "Cracking" 2FA

It's worth separating this from attacks that target the *code itself* — SIM-swapping to intercept an SMS, phishing the OTP in real time, or brute-forcing a 4-digit code. Those attacks are trying to obtain or guess the second factor.

This vulnerability doesn't touch the second factor at all. The code is never guessed, stolen, or intercepted — the attacker just walks around the entire second step, because the server never actually required it. It's the digital equivalent of a bouncer checking your ID at the front door, then waving you toward a second checkpoint that's supposed to scan your ticket — except the person at that second checkpoint never radios back to confirm you actually stopped there before you're already standing at your seat.

## Where Else to Look for This (Testing Checklist)

The "type the dashboard URL directly" trick is the classic version, but the same *root cause* — the server trusting a flow instead of enforcing a state — shows up in a few other places worth checking when testing a 2FA implementation:

- **Forced browsing** — after password login, try navigating straight to known "logged-in only" URLs before entering the code at all.
- **Response tampering** — if the app's verification-code page loads based on a value in an API response (e.g. `"mfaRequired": true`), try intercepting and flipping that value with a proxy tool. Some front ends trust that flag completely and simply won't show the code screen if it says `false`.
- **Session reuse** — check whether a session created before 2FA is completed stays valid indefinitely, even if you never finish the second step, close the tab, or hit "back."
- **Alternate entry points** — mobile apps, public APIs, and "magic link" email logins are sometimes built by a different team or added later, and may not enforce the same 2FA gate as the main website login form.
- **Password reset flow** — check whether resetting your password from a "forgot password" link drops you into an authenticated session without ever asking for the second factor.

---

# Part 2 — Hands-On Lab: 2FA Simple Bypass

This is a PortSwigger Web Security Academy lab, and it's a near-perfect real demonstration of everything in Part 1 — the "logged-in-only page" that never actually checks whether the second factor was completed.

**Given by the lab:**

| Account | Credentials |
|---|---|
| Your account | `wiener` : `peter` |
| Victim's account | `carlos` : `montoya` |

**Goal:** log into Carlos's account without ever seeing the 4-digit code that gets emailed to *him* — because you don't have access to his inbox.

## The Accidental Solve (and why it isn't the real lesson)

Before landing on the intended method, I stumbled onto a solve almost by accident: I went to the account page and logged straight in with the **victim's** credentials. It prompted for the 4-digit code as expected. I clicked the browser's back arrow, then clicked forward into the login page again — and a "My account" button appeared. Clicking it logged me straight into Carlos's account, code and all, and PortSwigger marked the lab as solved.

That did technically work, but it's worth being honest about *why* it's not the version worth learning from: it depends on the browser re-rendering a page from navigation history rather than on a clean, repeatable request you deliberately crafted. It happens to hit the exact same underlying flaw (the account page never actually checks whether 2FA was completed for that session), but it's not something you could reliably reproduce in a report to a real client — "click back twice and hope a button appears" isn't an exploit write-up. The version below is the one that actually demonstrates *why* the bug exists and how to prove it on demand.

## The Intended Solve — Step by Step

### Step 1 — Log into your own account first

Log in normally with your own credentials, `wiener` / `peter`. As expected, the app asks for a 4-digit verification code sent to your email.

Since this is a lab environment, PortSwigger gives you a built-in "Email client" button to view that inbox without needing a real mail server.

![Email Client Button](Images/2email_client.png)

Open it and grab the code:

![2FA 4 digit code](Images/2fa_code.png)

### Step 2 — Enter the code and inspect the resulting URL

After submitting the correct code for your own account, look at the address bar. It reads:

```
/my-account?id=wiener
```

![URL My account and id wiener as username](Images/2URL_myaccount_wiener.png)

This is the important clue: the page you land on *after* successfully completing 2FA is just `/my-account`, and it happens to carry an `id` parameter identifying whose account to display. Nothing about that URL screams "and 2FA was verified" — it's a plain page load.

### Step 3 — Log out, and start logging in as the victim

Log out, then log back in with the victim's credentials, `carlos` / `montoya`. The app accepts the password and — same as before — shows the "enter your 4-digit code" screen (served from a URL like `/login2`). This time, you have no way to read that code, because it was emailed to Carlos's inbox, not yours.

### Step 4 — Forced-browse straight to `/my-account` instead

This is the actual bypass. Instead of entering any code, edit the URL directly: remove `/login2` and replace it with `/my-account`.

![Bypassing 2FA](Images/2bypassing_2FA_with_my-account.png)

The page loads — fully authenticated as Carlos, without ever touching his 4-digit code. I also appended `?id=carlos` out of habit (mirroring the pattern from Step 2), but it turned out not to be necessary — `/my-account` alone was enough. That's worth sitting with for a second: the `id` parameter isn't what's granting access here. The account page was always going to serve *whichever* session cookie you're currently holding; the parameter just controls what gets displayed, not who you're allowed to be. The actual vulnerability is entirely about `/my-account` never checking for a completed second factor — full stop.

## Tying It Back to Part 1

This lab is a textbook instance of the pattern from Part 1: the server marked the session `authenticated=true` the moment the password check passed, and `/my-account` only ever checked *that* flag. It never checked whether the follow-up code screen (`/login2`) had actually been completed. Whether you reach that unprotected page by carefully editing the URL (Step 4) or by accident via the browser's back button (the "accidental solve" above), you're exploiting the exact same gap — a page that should require `authenticated AND mfa_verified`, but only checks `authenticated`.

## Key Takeaways

- 2FA that's enforced only by *redirecting the user* to a code page is not actually enforcing anything — it's a UI convention the server should treat as a suggestion at best.
- The fix is simple in principle: every protected page and API endpoint must check a distinct `mfa_verified` state, separate from the basic `authenticated` state, and refuse to serve data until both are true.
- This bug class is dangerous specifically because it requires **zero interaction with the second factor itself** — no phishing, no SIM-swap, no guessing. An attacker with just a stolen password can walk straight past a 2FA screen that looks, to a human, completely secure.
- When you find a bypass, it's worth distinguishing an *accidental*, browser-behavior-dependent path from a *deliberate*, reproducible one (like directly forced-browsing to the protected URL) — only the latter makes for a credible, reportable finding.
