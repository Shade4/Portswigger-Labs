# Username Enumeration via Subtly Different Responses — Full Detailed Walkthrough

> **Lab:** PortSwigger Web Security Academy — *Username enumeration via subtly different responses*
> **Category:** Authentication — Broken Brute-Force Protection / Username Enumeration
> **Tools:** Browser, Burp Suite (Intruder)
> **Goal:** Enumerate one valid username, brute-force that user's password, and log in to their account.

This is an expanded, step-by-step version of the original notes, with every request, every response, and every Burp Suite setting explained in detail — plus a look at how this exact bug class shows up in real production software.

---

## 1. What This Lab Is Actually Teaching

Most login forms are built to say something generic like *"Invalid username or password"* no matter what goes wrong, specifically so an attacker can't tell whether they guessed a real account or not. That's good practice — **but good practice only helps if it's implemented consistently.**

In this lab, the developer clearly *intended* to return one uniform error message. Functionally, they succeeded — every invalid attempt shows what looks like the same sentence. The flaw is that the message wasn't generated from a single shared code path. Somewhere in the application there are (at least) two near-identical string templates:

- one used when the **username itself doesn't exist**
- one used when the **username exists but the password is wrong**

Because those two messages were typed out separately by a human at some point, they drifted apart by a single character — a trailing period in one, a trailing space in the other. Byte-for-byte, the responses are no longer identical, even though they render as visually indistinguishable text in the browser. That single-byte drift is the entire vulnerability.

This is the core lesson: **"identical" and "byte-identical" are not the same thing**, and tools like Burp Intruder exist precisely to compare hundreds of responses at the byte level, something no human clicking through a browser would ever notice.

---

## 2. Environment & Tools

| Tool | Purpose |
|---|---|
| Browser (with Burp's certificate trusted) | Load the lab, submit the login form once to generate a baseline request |
| Burp Suite, Proxy → Intercept | Capture the raw HTTP request the browser sends |
| Burp Suite, Intruder | Automate sending that request hundreds of times with one field swapped out per request |
| Candidate username list | ~150 common usernames provided by the lab |
| Candidate password list | ~100 common passwords provided by the lab |

Burp sits as a proxy between your browser and the lab server. Every request your browser makes passes through Burp first, which is what lets you pause, inspect, and modify traffic before it's forwarded on — and later, resend a modified version of that same request automatically, many times over, with Intruder.

---

## 3. Step-by-Step Walkthrough

### Step 1 — Reconnaissance: Find the Login Form

Open the lab and go to **My account**. Unauthenticated, this redirects you to `/login`, which renders a simple HTML form with two fields, `username` and `password`, and a submit button.

At this stage you're just establishing what a "normal" request looks like — you need one real request to capture before you can start automating.

![testing login](Images/2testing_login.png)

The screenshot above shows the login page after typing `test` / `test` into both fields, purely to generate traffic Burp can capture. It doesn't matter that these credentials are wrong — the point of this request is only to see the *shape* of the traffic.

### Step 2 — Intercept the Request

With Burp's **Intercept** switched **on** in the Proxy tab, click **Log in**. The browser sends its request, but Burp holds it before it reaches the server, showing you the raw HTTP request. It looks roughly like this:

```
POST /login HTTP/2
Host: 0a1b00c1234567890abcdef.web-security-academy.net
Cookie: session=Xk92n1Qp...
Content-Length: 27
Content-Type: application/x-www-form-urlencoded
...

username=test&password=test
```

What each part means:

- **`POST /login`** — the method and path. `POST` is used (rather than `GET`) because the form is submitting sensitive data that shouldn't sit in the URL or browser history.
- **`Host`** — which lab instance you're talking to (every PortSwigger lab gets a unique random subdomain).
- **`Cookie: session=...`** — a pre-authentication session token the server issued when you first loaded the page. It's how the server tracks your login attempt across requests, even before you're actually logged in.
- **`Content-Type: application/x-www-form-urlencoded`** — tells the server the body is standard `key=value&key=value` form data, not JSON.
- **The body, `username=test&password=test`** — the two values you typed, sent exactly as-is.

Right-click this intercepted request and choose **Send to Intruder**. This copies the entire request over to the Intruder tab, where you can configure an automated attack against it — the raw traffic itself never changes just because you sent it to Intruder; Intruder only re-sends *modified copies* of it.

### Step 3 — Choose the Attack Type and Mark the Username Position

In Intruder's **Positions** tab, you'll see the same raw request, but Burp auto-highlights values it thinks look like parameters using `§` markers, e.g. `username=§test§&password=§test§`.

Click **Clear §** to remove all of them — you want full manual control. Then double-click just the word `test` in `username=test`, and click **Add §** (or use the keyboard shortcut). Now only the username value is wrapped in markers:

```
username=§test§&password=test
```

![username payload position](Images/2username_payload_position.png)

For **Attack type**, leave it on **Sniper**. Burp offers four attack types, and the difference matters:

| Attack type | Behavior |
|---|---|
| **Sniper** | One payload list, fired through one position at a time. Best for testing a single field in isolation — exactly our case. |
| Battering ram | Same payload inserted into *all* marked positions simultaneously (same value everywhere). |
| Pitchfork | Multiple payload lists, one per position, stepped through in lockstep (item 1 with item 1, item 2 with item 2...). |
| Cluster bomb | Multiple payload lists, one per position, tried in every combination (a full cross-product). |

Since we're only varying `username` while holding `password` fixed at `test`, Sniper is the correct choice — we're not trying to find a *working* login yet, only trying to find which username produces a *different-looking failure*.

### Step 4 — Load the Username Wordlist

Switch to the **Payloads** tab. Payload type is **Simple list**. Paste in the full list of ~150 candidate usernames the lab provides (`carlos`, `root`, `admin`, `test`, `guest`, ...). Each entry in this list will be substituted, one at a time, into the `§...§` marker from Step 3, and fired as a separate HTTP request.

### Step 5 — Configure Grep - Extract (the key step)

Here's the problem: if you just ran the attack right now and looked at Burp's default columns (HTTP status code, response length), **every single response would look identical** — same `200 OK`, same byte length. That's by design; this lab is specifically built so the obvious signals don't work, forcing you to look at the actual response content.

This is what **Grep - Extract** is for. It lets Burp pull a specific snippet out of *every* response in the attack and display it as its own sortable column, so you can visually scan hundreds of results at once instead of opening each response individually.

To configure it:

1. Go to the **Settings** tab of the Intruder attack (in older Burp Suite versions this was called **Options**) and find the **Grep - Extract** section.
2. Click **Add**, then **Fetch response** — Burp sends one sample request (using the base value, `test`) so you have a real response to select from.
3. In that sample response, find the error paragraph:

   ```html
   <p class=is-warning>Invalid username or password.</p>
   ```

4. Highlight **only** the inner text, `Invalid username or password.`, and click **OK**.

From now on, every response in the attack will have that exact byte range extracted and shown in its own column — letting you compare a couple of words per row instead of scrolling through full HTML pages 150 times.

![adding payload position in password](Images/2payload_position_password.png)

*(This screenshot is referenced again in the original notes when configuring the password stage — see Step 8 below.)*

### Step 6 — Run the Username Attack

Click **Start attack**. Burp fires ~150 requests, one per candidate username, and populates the results table — including your new Grep - Extract column.

Scroll (or click the column header to sort) down that column. Almost every row will read:

```
Invalid username or password.
```

...with a trailing period. But one row will be subtly different:

```
Invalid username or password 
```

![change in warning in username enumeration](Images/2change_in_warning_username.png)

No period — and if you look even closer, a trailing space where the period would have been. That row's username is your enumerated valid account.

**Why does this happen?** Somewhere on the server, the logic likely looks conceptually like this:

```
if user does not exist:
    return "Invalid username or password."      # generic path
elif user exists but password is wrong:
    return "Invalid username or password "       # a *different* string literal, missing the period
```

Two separate developers (or the same developer on two separate occasions) wrote what was supposed to be the same message twice, and the copies weren't kept in sync. That's an extremely common, very realistic mistake — which is exactly why this lab exists.

### Step 7 — Repoint the Payload at the Password Field

Go back to the original intercepted request in the Positions tab (or send the base request to Intruder again). This time, hard-code the username you just found and mark the password instead:

```
username=<found_username>&password=§test§
```

### Step 8 — Load the Password Wordlist

In the Payloads tab, **clear** the old username list and paste in the ~100 candidate passwords instead. Attack type stays **Sniper** — again, one field varying, one field fixed.

![password found using brute force](Images/2passowrd_brute_force.png)

Your Grep - Extract configuration from before carries over automatically, since it's tied to the exact byte-string `Invalid username or password.`, not to a specific field.

### Step 9 — Run the Password Attack and Read the Result

Start the attack. This time, watch the Grep - Extract column again:

- For every **wrong** password, the row shows:
  ```
  Invalid username or password
  ```
  (no trailing period this time, because we now have the *correct* username — we're on the "user exists" code path for every single attempt.)

- For the **one correct** password, the Grep - Extract column comes back **empty**.

That's the tell. An empty extraction means Burp searched the response for the literal text `Invalid username or password.` and didn't find it anywhere — because on a successful login, the server doesn't render that warning paragraph at all. It instead returns a redirect (or a completely different page) straight to the account dashboard. No error text exists in that response for Burp to extract, so the column is blank.

> **Pro tip, for next time:** you can often catch this same signal without configuring Grep - Extract at all, just by re-enabling the **Status code** and **Length** columns for this second attack — a successful login is very likely to return a `302 Found` redirect and a different response length than every `200 OK` failure around it. Grep - Extract was necessary in the *username* stage because the lab deliberately kept status codes and lengths identical there; it's usually optional once you've already reached the password stage, but it's a reliable habit to keep either way.

### Step 10 — Log In

Take the username from Step 6 and the password from Step 9, enter them in the actual login form in your browser (Intercept switched off now), and submit. You land on `/my-account`, and the lab is marked as solved.

---

## 4. A Fully Annotated Request/Response Example

To make the byte-level difference completely concrete, here's what the three relevant request/response pairs look like side by side (values illustrative, structure accurate to the lab):

**Request A — wrong username, wrong password:**
```
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 27

username=zzzz&password=test
```
**Response A:**
```
HTTP/2 200 OK
Content-Length: 3143

...
<p class=is-warning>Invalid username or password.</p>
...
```

**Request B — correct username, wrong password:**
```
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 29

username=carlos&password=test
```
**Response B:**
```
HTTP/2 200 OK
Content-Length: 3143

...
<p class=is-warning>Invalid username or password </p>
...
```

Notice `Content-Length` is *identical* in both — the lab authors padded things so that signal is a dead end. The only difference is the single trailing character inside the `<p>` tag: `.` versus a space. That's a 0-byte-length difference in practice (a period and a space are both one byte), so even response length comparison at the raw byte level wouldn't catch it here — only comparing the *content* does.

**Request C — correct username, correct password:**
```
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 34

username=carlos&password=montana
```
**Response C:**
```
HTTP/2 302 Found
Location: /my-account
Content-Length: 0
```
No warning paragraph exists at all — the whole response shape changes.

---

## 5. Real-World Examples (This Isn't Just a Lab Trick)

This class of bug — a login or account-related endpoint leaking whether a user exists through *some* inconsistency, even when the visible error text looks the same — shows up in real, shipped software regularly. Two examples, at opposite ends of the technology timeline:

### Historical example: Apache `mod_userdir`

Older Apache HTTP Server configurations that exposed personal home directories at URLs like `http://example.com/~username` had a well-documented enumeration flaw: requesting a username that existed on the system but had no public `public_html` directory returned an HTTP 403 response (a "you don't have permission" message), while requesting a username that didn't exist at all returned an HTTP 404 (a "not found" message). The rendered text was different, but more importantly, so was the raw **status code** — 403 vs. 404 — letting an attacker enumerate valid system usernames purely by watching which code came back, with no need to inspect page content at all.

### Recent example: Budibase account lockout (CVE-2026-73306)

A 2026 vulnerability disclosed in the open-source low-code platform Budibase followed almost the exact same pattern as this lab, just implemented through account lockout instead of a raw error string. The login endpoint locked an account after five failed attempts — but the lockout counter was only ever incremented for emails that actually existed in the database. So after five failed logins:

- an **existing** email flipped to a distinct `403` response carrying an `X-Account-Locked: 1` header, a `Retry-After: 900` header, and the message "Account temporarily locked"
- a **non-existent** email kept returning a generic `403 "Unauthorized"` forever, no matter how many attempts were made

An attacker could fire five or six requests per candidate email and simply watch for that header to appear — a Grep-Extract-style check on `X-Account-Locked`, exactly like extracting the warning text in this lab — to build a list of confirmed accounts at high speed, with the added side effect of locking real users out of their own accounts for 15 minutes as a denial-of-service bonus. The fix, as is almost always true for this bug class, was to make the lockout/failure logic run identically whether or not the account exists, so the two code paths can no longer diverge.

### The general pattern to watch for

In real audits, this kind of subtle differential shows up far more often on **password-reset** and **account-creation** forms than on login forms directly, because developers are more inclined to explicitly confirm "an email was sent" on those flows. The differences to check for, beyond visible text, include:

- **HTTP status code** (200 vs 302, 200 vs 403/404)
- **Response headers** (a lockout header, a "set-cookie" that only appears for real accounts, a different `Content-Length`)
- **Response timing** (checking a password hash for a real user is often measurably slower than short-circuiting immediately for a nonexistent one)
- **Redirect target** (redirected to a "check your email" page vs. staying on the same form)
- **Whitespace, punctuation, or capitalization drift** between two near-identical hardcoded strings — precisely what this lab demonstrates

---

## 6. Why This Actually Matters (Impact Chain)

Username enumeration is rarely dangerous by itself — but it's almost always **step one** of a longer attack:

1. **Enumerate** — confirm which usernames/emails are real accounts (this lab, Step 6).
2. **Credential stuff / brute-force** — now that effort isn't wasted guessing usernames that don't exist, run common or breached passwords against every *confirmed* account (this lab, Step 9).
3. **Account takeover** — a successful password match hands over the account outright.
4. Even without cracking the password, a confirmed list of valid accounts is directly useful for **targeted phishing** (attackers know exactly who to impersonate a "password reset" email to) and for **credential-stuffing other services** with the same email, since people frequently reuse passwords across sites.

---

## 7. How Developers Should Fix This

- Return the **exact same string, status code, headers, and timing** regardless of whether the username exists — generate it from a single shared code path, never two separately-written templates.
- Apply **rate limiting and lockout using the submitted identifier itself**, not conditioned on whether that identifier matched a real account (this is precisely the mistake in the Budibase case above).
- Add a **deliberate, constant delay** (or a dummy password-hash comparison) on the "user not found" path, so it takes roughly as long as the "user found, wrong password" path.
- On password-reset and signup flows specifically, always show a generic "if an account exists, an email has been sent" message rather than confirming existence either way.
- Rate-limit and CAPTCHA the login endpoint itself, so even a perfectly consistent response can't be brute-forced at scale from a single IP.

---

## 8. Key Takeaways

- "The error message looks the same" is not proof that it *is* the same — always compare at the byte/content level, not by eye.
- When the obvious differentiators (status code, response length) are deliberately or accidentally normalized, **Grep - Extract** is the tool that lets you compare arbitrary response content across a whole attack at once.
- A two-stage Sniper attack (enumerate the field first, then brute-force the second field against the confirmed value) is a standard, reusable pattern — not just for logins, but for any two-parameter endpoint where one value gates the other.
- This isn't an academic-only bug: the same root cause (two code paths that were supposed to produce identical output, but don't) has shown up in real software from 1990s Apache configs to 2026-era SaaS platforms.

---

## References (further reading)

- PortSwigger Web Security Academy — Authentication topic (background on this lab category)
- Apache `mod_userdir` enumeration behavior: https://docs.kentico.com/k8/securing-websites/developing-secure-websites/enumeration
- CVE-2026-73306 / GHSA-cr7p-cr3q-h5cm — Budibase account enumeration via login lockout response differential: https://corgea.com/advisories/vulnerabilities/CVE-2026-73306
- General overview of user enumeration on login, password-reset, and signup forms: https://www.vaadata.com/blog/?p=1655
