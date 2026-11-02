# Password Reset Poisoning

## Background: Resetting Passwords via a URL

A common way to implement "forgot password" functionality is for the server to email the user a unique link containing a reset token, e.g.:

```
https://example.com/reset-password?token=a0b1c2d3e4f5
```

The user clicks the link, the app verifies the token against the one it stored for that user, and if it's valid and unexpired, lets them set a new password. This is safe *as long as the domain portion of that link is always generated from a trusted, server-side value* — not from anything the client sent in the request.

## What Password Reset Poisoning Is

Password reset poisoning happens when the server builds that reset link **dynamically**, using a value taken from the incoming HTTP request — most often the `Host` header — instead of a hardcoded or config-defined domain. Something like:

```python
reset_link = f"https://{request.headers['Host']}/reset-password?token={token}"
```

An attacker doesn't need access to the victim's inbox or browser to abuse this. They just need to trigger a password reset request for the victim's email address while sending a modified `Host` header (or an override header like `X-Forwarded-Host`, `X-Forwarded-Server`, or `X-Original-Host`, which some frameworks also trust):

```
POST /forgot-password HTTP/1.1
Host: attacker-controlled.com
...
email=victim@example.com
```

If the backend trusts that header, the email that lands in the victim's real inbox now contains a legitimate-looking reset link — but pointing at the attacker's domain:

```
https://attacker-controlled.com/reset-password?token=a0b1c2d3e4f5
```

The attacker doesn't even need the victim to notice anything suspicious in the link's *path* — only the domain is poisoned, and most people don't scrutinize that closely. When the victim clicks it, their browser sends the valid reset token straight to the attacker's server (via the query string or server logs). The attacker now holds a working token for the victim's account and can use it against the *real* site to set a new password — completing the takeover without ever touching the victim's credentials directly.

It's a textbook example of why **trusting user-controllable input (headers included) to build security-sensitive links or logic is dangerous** — the request is attacker-authored even when it looks like routine traffic.

## Real-World Example

An older version of **Piwik** (now known as Matomo, the open-source web analytics platform) was documented as vulnerable to exactly this issue: it generated password reset emails using the `Host` header from the incoming request instead of a fixed, trusted domain. An attacker could submit a password reset request for a victim's account with a forged `Host` header pointing at a server they controlled. The victim received a real reset email from the real application, but the link inside it pointed to the attacker's domain — handing over the reset token the moment the victim clicked it. This case is often used as the canonical demonstration of Host header–based password reset poisoning, because it shows the full chain end-to-end: a trusted-looking email, a single manipulated header, and a complete account takeover with no interaction needed from the victim beyond clicking "reset my password."

## Lab Walkthrough: Password Reset Poisoning via Middleware

**Scenario:** You control your own account (`wiener:peter`). The victim, `carlos`, will click any link that lands in his inbox. Goal: take over Carlos's account by poisoning the password reset link he receives, without ever seeing his real email or password.

**Setup:** Burp Suite running, the browser routed through it via FoxyProxy, lab loaded.

1. **Find the reset flow.** On the "My account" page there's a "Forgot password?" link. Clicking it opens a form asking for a username or email. This is the entry point for the whole attack — triggering a reset email for *any* account requires no authentication of your own.

   ![pass reset functionality](Images/Lab%2011/pass_reset_funtionality.png)

2. **Capture the baseline request.** With Burp's intercept on, submit the form with your own username. Send the resulting `POST /forgot-password` request to Repeater rather than letting it pass through, so it can be replayed and edited freely:

   ```
   POST /forgot-password HTTP/1.1
   Host: 0a...web-security-academy.net
   ...
   username=wiener
   ```

   ![pass reset request](Images/Lab%2011/pass_reset_request.png)

3. **Spot the trust gap.** This app's `/forgot-password` handler sits behind middleware written for reverse-proxy deployments: if the app is ever placed behind a load balancer or CDN that rewrites the `Host` header to an internal address before forwarding the request, the middleware is meant to fall back on an `X-Forwarded-Host` header — a header a *legitimate* reverse proxy sets to tell the backend what domain the client actually requested — to reconstruct the public-facing domain for things like reset links. The bug: the middleware accepts that header from *anyone*, with no check on whether it actually came from the trusted proxy or from a browser talking to the app directly.

4. **Exploit it.** Open the "Go to exploit server" page and note your personal exploit-server URL. Back in Repeater, add `X-Forwarded-Host` pointing at that URL, and change `username` to `carlos`:

   ```
   POST /forgot-password HTTP/1.1
   Host: 0a...web-security-academy.net
   X-Forwarded-Host: exploit-...exploit-server.net
   ...
   username=carlos
   ```

   Send it. The app builds Carlos's reset link using your exploit server as the "host," instead of the real domain.

   ![forwarding request to carlos with exploit server](Images/Lab%2011/forward_to_carlos_with_exploit_server.png)

5. **Harvest the token.** Carlos auto-clicks whatever lands in his inbox, so once the poisoned email "arrives," he follows the link straight to your exploit server. Open the exploit server's **Access log** and find the `GET /forgot-password?temp-forgot-password-token=...` entry — that query string is Carlos's real, valid reset token, now sitting in your logs instead of his inbox.

   ![carlos pass reset token](Images/Lab%2011/carlos_pass_reset_token.png)

6. **Replay the token against the real site.** The token is only meaningful on the actual application, not on your exploit server, so copy it and splice it into the genuine reset-password URL (from your own reset email) in place of your own token:

   ```
   https://0a...web-security-academy.net/forgot-password?temp-forgot-password-token=CARLOS-STOLEN-TOKEN
   ```

   ![before changing URL](Images/Lab%2011/before_chaning_link_to_forward_carlos_pass_reset.png)

   ![after changing URL](Images/Lab%2011/after_chaning_link_to_forward_carlos_pass_reset.png)

8. **Set a new password.** Loading that URL drops you straight on Carlos's "set new password" page — the backend only checks whether the token is valid for *some* account, not whether you're the person it was issued to, so it can't distinguish this from Carlos legitimately following his own link.

   ![new pass carlos](Images/Lab%2011/new_pass_enter_screen.png)

9. **Log in as `carlos`** with the password you just set. Lab solved.

## Why This Vulnerability Exists

This variant is specifically about **middleware that's doing exactly what it was designed to do — just without checking who it's talking to.**

In production, apps are very often deployed behind infrastructure (a load balancer, a CDN, an API gateway) that terminates the client's connection and then forwards the request onward over its own internal network. That forwarding step frequently rewrites the `Host` header to an internal hostname or IP, since that's what the backend actually needs to route the request internally. The problem this creates: if the backend ever needs to reconstruct the domain the *original* client actually typed or clicked — exactly the case for building an absolute URL like a password reset link — the real `Host` header is gone.

`X-Forwarded-Host` exists to solve that: a well-behaved reverse proxy sets it to the original host *before* rewriting `Host` and forwarding the request, so the backend can recover the client-facing domain. That's a completely legitimate, common pattern — the middleware in this lab isn't buggy for reading `X-Forwarded-Host`, it's buggy because it reads it **unconditionally**, from **any** request, instead of only when the request genuinely passed through the trusted proxy.

Since HTTP headers are just text a client sends, there's nothing stopping an attacker from connecting to the app directly (or through the real proxy, which will happily pass along a header it didn't itself set, if it isn't configured to strip or overwrite one) and setting `X-Forwarded-Host` themselves. The backend has no way to tell "the proxy set this" from "the attacker typed this" — it trusts the header's mere presence as proof of where the original request came from, which is precisely the gap that lets the reset link get redirected to an attacker-controlled domain.

## Why It Matters

- The victim gets a completely genuine email from the real sending address — no phishing domain, no spoofed sender. Only the link's destination is wrong.
- It bypasses defenses aimed at phishing or credential theft, since the user never enters anything on a fake login page — they just click a link.
- It can chain with other bugs (e.g., an open redirect, or a reset token that's valid even if unused by the real user) to become fully automatable at scale.

## Prevention

- Never derive security-sensitive URLs (reset links, email verification links, etc.) from `Host`, `X-Forwarded-Host`, or any other client-controlled header.
- Use a server-side allowlist or hardcoded config value for the application's canonical domain.
- Bind reset tokens to additional context (e.g., invalidate on first use, short expiry, optionally tie to session/IP) so a leaked token has minimal blast radius.
- If multiple domains legitimately need support (multi-tenant apps), validate the `Host` header against an explicit allowlist rather than trusting it outright.
- If the app genuinely sits behind a reverse proxy that needs to forward the original host, configure that proxy to **strip or overwrite** any `X-Forwarded-Host` a client sends before adding its own, and configure the app/framework to trust forwarded headers only when the request's source IP matches the proxy itself — not merely because the header is present.
