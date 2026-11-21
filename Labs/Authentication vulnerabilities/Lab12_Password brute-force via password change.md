# Changing User Passwords: Vulnerabilities in Password Change Functionality

> Part of my notes on PortSwigger Web Security Academy's **"Vulnerabilities in other authentication mechanisms"** topic, finished off with a full walkthrough of the **"Password brute-force via password change"** lab.
> For learning and authorized testing only (labs, bug bounty programs, or systems you have written permission to test).

---

## TL;DR

A "change password" page looks harmless, but underneath it has to answer the same question a login page does: *"Is this person really who they say they are?"* If it answers that question carelessly, an attacker can use it as a **second, weaker login form** and brute-force or enumerate accounts through it. If it lets the **client decide whose password is being changed**, it can become a direct route to taking over *any* account.

---

## What is it?

Most sites let a logged-in user change their password by submitting a form with three things:

1. their **current password**
2. their **new password**
3. the **new password again** (confirmation)

Behind the scenes the server has to check that the current password actually belongs to the account being modified. That check is essentially the same logic as the login check, so it inherits the same weaknesses.

---

## Why it's dangerous

### 1. It's a login form in disguise

Because the page verifies "current password = correct?", an attacker who can reach it can guess passwords through it. Everything you'd try against a login page applies here too:

- **Username enumeration:** different error messages for different failures reveal valid usernames or passwords.
- **Brute-forcing:** if the change-password page doesn't have the same lockout or rate limiting as the login page, it becomes an easier target.
- **Response oracles:** the order of the server's checks can leak information. For example, many implementations verify the current password *first* and the "do the two new passwords match?" check *second*. That means the attacker can send two deliberately mismatched new passwords and read the response:
  - "Current password is incorrect" → the guess was wrong
  - "New passwords do not match" → the guess was **right** (the server got past the first check)

  The attacker never needs to successfully change anything, they just read the error message.

### 2. The server trusts a client-supplied identity

The scarier case is when the request itself says *whose* password is being changed, for example through a hidden form field:

```http
POST /change-password HTTP/1.1
Host: shop.example
Content-Type: application/x-www-form-urlencoded

username=alice&current-password=hunter2&new-password-1=Passw0rd!&new-password-2=Passw0rd!
```

If the server uses that `username` value directly (instead of looking up the user from the authenticated session), the attacker can:

- edit `username=alice` to `username=bob` and target any account they want, and
- in the worst case, reach the endpoint **without being logged in at all**, so the whole thing can be scripted at scale (enumerate usernames, then brute-force their passwords).

---

## Attack flow (simplified)

```
Attacker                               Server
   |                                      |
   |-- POST /change-password ------------>|
   |   username=bob (tampered)            |
   |   current-password=<guess>           |
   |   new-password-1 != new-password-2   |
   |                                      |
   |<-- "Current password is incorrect" --|   -> wrong guess, try the next one
   |<-- "New passwords do not match" -----|   -> correct guess found
```

Repeat with a wordlist (Burp Intruder is the usual tool for this) and watch for the one response that differs. The lab walkthrough below is exactly this attack, end to end.

---

## Lab Walkthrough: "Password brute-force via password change"

**Setup:** PortSwigger Web Security Academy lab, solved with Burp Suite.
**Your own account:** `wiener` / `peter`
**Target (victim) account:** `carlos`
**Goal:** recover Carlos's password through the password-change form, then log in as him.

The two weaknesses from the explanation above — a **client-controlled identity field** and a **response oracle** in the error messages — are exactly what this lab is built around. Here's how each step uncovers and then exploits them.

### Step 1 — Log in with your own credentials

Log in as `wiener` with the password `peter`. You need an authenticated session before the password-change form is even reachable, and you want to see the feature behave normally as a legitimate user before trying to abuse it.

![Logging in with the given credentials](Images/Lab%2012/login_with_given_credentials.png)

### Step 2 — Open "My account" and experiment with the password-change form, with Burp running

Go to **My account**. Below the email field is the password-change form: **Current password**, **New password**, **Confirm new password**, **Change password**.

![The My account page with the password-change form](Images/Lab%2012/access_the_password_change_functionality_and_enperiment_with_it.png)

With Burp's proxy capturing traffic, submit the form once with any values and look at the raw request. The form visually shows only three fields, but the POST body actually carries a fourth parameter:

```
username=wiener&current-password=...&new-password-1=...&new-password-2=...
```

**Why this matters:** `username` isn't something the page asks you to type — it's a *hidden* input, pre-filled server-side with whoever is currently logged in, and sent along silently. The page's own design already tells you how the server decides which account to check the password against: it reads it from a request parameter. A server that does this instead of looking up the logged-in user from the session has effectively handed identity-selection over to the client. That single design choice is what the rest of the attack depends on — nothing has been exploited yet, but the hook has been found.

### Step 3 — Work out the server's check order from the error messages

Try a few combinations and read the responses carefully:

| current-password | new-password-1 vs new-password-2 | Response |
|---|---|---|
| wrong | match each other | account temporarily locked |
| wrong | **don't** match | `Current password is incorrect` |
| **correct** | don't match | `New passwords do not match` |

This tells you the server's logic runs in this order:

1. Is the account locked out from too many recent failures?
2. Does `current-password` match the real password? If not → `"Current password is incorrect"`, stop here.
3. Do `new-password-1` and `new-password-2` match each other? If not → `"New passwords do not match"`.
4. Only if both checks pass does it actually update the password.

The important detail is that **step 2 runs before step 3**. Seeing `"New passwords do not match"` is only possible if the server already got past the current-password check — so that message is proof the guess was correct, even though nothing was ever actually changed. Submitting two new passwords that deliberately don't match turns this into a safe, repeatable oracle: you can hammer the endpoint with password guesses all day, and it will never lock you out for "too many successful changes" or alert the real user, because step 4 never runs.

### Step 4 — Capture a password-change request and send it to Intruder

In the form, submit your real current password along with two different values for the new password (so the request is guaranteed to stop at step 3, not step 4), capture it in Burp, and send it to Intruder.

![Captured request showing the hidden username parameter](Images/Lab%2012/captured_request_of_change_password_functionality_and_send_it_to_the_intruder.png)

This is the same request from Step 2, now sitting in Intruder — body parameters and all, including `username`, right there in plain text and fully editable. Manually resending guesses one at a time would work in principle but is far too slow for a real wordlist; Intruder automates "send this same request repeatedly, changing one value, and record every response" so hundreds of guesses can be tried in seconds.

### Step 5 — Retarget the request at Carlos, and mark the password field as the payload position

On the **Positions** tab:

- Edit the request body directly and change `username=wiener` to `username=carlos`.
- Clear Burp's auto-selected positions, then highlight only the `current-password` value and mark it as a payload position (`§...§`).
- Leave `new-password-1` and `new-password-2` as two fixed, different literal values — they must never match, so the actual password change never fires.

![Intruder payload position and password list configuration](Images/Lab%2012/payload_configurations.png)

This is the pivot from "testing my own account" to "attacking Carlos's." Because the server trusts the `username` field instead of deriving identity from the session cookie, every request Intruder now fires is really asking the server: *"Does this guess match Carlos's current password?"* — and the server will dutifully answer, even though the attacker was never authenticated as Carlos.

Only `current-password` is marked as the variable. Keeping exactly one moving part per request means every response can be attributed to that one guess (a classic Sniper attack), and keeping the two new-password values fixed-but-different means nothing on Carlos's account ever actually gets changed while you're probing it.

### Step 6 — Supply the password list

In the **Payloads** panel, choose **Simple list** as the payload type and load (or paste) a candidate password list.

*(Same screenshot as Step 5 — the Payloads panel is visible on the right.)*

In a real assessment this would be a proper wordlist (e.g. `rockyou.txt`); for the lab, a short common-password list is enough because the lab seeds one specific password for Carlos.

### Step 7 — Flag the differentiating response, then start the attack

Open **Settings → Grep - Extract**, select the warning text in a sample response (e.g. `Current password is incorrect`), and let Burp define the start/end markers around it. This adds a new column to the results grid that shows, for *every* request, exactly what message came back at that spot in the page.

![Configuring Grep - Extract on the warning message](Images/Lab%2012/grep_extract_use.png)

A plain Grep - Match rule only gives you a yes/no flag for one fixed string. Grep - Extract instead prints the *actual* message text for every row in its own column — so both `"Current password is incorrect"` and the rarer `"New passwords do not match"` show up side by side without needing two separate rules, and the one row that reads differently from all the rest jumps out immediately. Click **Start attack**.

### Step 8 — Find the one response that reads differently

When the attack finishes, scan the new results column. Every row shows the same generic `"Current password is incorrect"` — except one, which shows `"New passwords do not match"`. That payload value is Carlos's real current password.

![Attack results — one response differs from all the rest](Images/Lab%2012/got_the_password_from_brute_force_login_with_it_to_solve_the_lab.png)

In this run, payload `159753` on row 72 is the outlier: its response is 4 bytes shorter and arrived carrying the other message. That's the response oracle from Step 3 paying off — the server only ever reaches the "do the new passwords match" check (and therefore only ever produces that message) for the one guess that got past the current-password check first.

### Step 9 — Log out, and log back in as Carlos with the recovered password

The endpoint only ever *told you* whether a guess was correct — it never logged you in and never changed anything. Log out of `wiener`, then log in with username `carlos` and the password identified in Step 8. This is the step that actually turns "I can tell a correct password from an incorrect one" into "I'm authenticated as the victim" — i.e. where the vulnerability becomes a real account takeover rather than just an information leak.

### Step 10 — Click "My account" to solve the lab

The account page now loads under Carlos's identity, confirming the takeover and solving the lab.

### Why this worked, in one sentence

A hidden, client-editable `username` field let the attacker point the password check at someone else's account, and an error message that leaked *which* check had failed turned that check into a brute-forceable oracle — and because the endpoint never actually had to succeed to leak that signal, none of the login page's usual brute-force protections ever triggered.

---

## Real-world example: GitLab, CVE-2023-7028 (2023 to 2024)

**Important note on scope:** this one is a *password reset* flaw rather than a *password change* form. I'm using it because it's the best-documented real-world case of the **same root cause**: the server lets attacker-controlled input decide *whose account* a password operation applies to (or where its secret goes), without proper verification.

**What happened**

- GitLab 16.1.0 (May 2023) added the ability to send password reset emails to a **secondary email address** on an account.
- The reset request accepted the email field as a **list**. An attacker could submit a reset request for a victim's address and put a **second, attacker-controlled address** in the same list.
- GitLab sent the reset email to **both** addresses without verifying that the second one belonged to the victim. The attacker just clicked the link in their own inbox and set a new password.
- It was **zero-click** for the victim (no interaction needed). Accounts protected by two-factor authentication couldn't be fully taken over this way, though attackers could still trigger resets.
- It was rated **CVSS 10.0 (critical)**. It was reportedly found in December 2023 and paid a $35,000 bounty. Patches shipped in January 2024 (16.7.2, 16.6.4, 16.5.6, plus backports to older branches).
- It later landed on **CISA's Known Exploited Vulnerabilities** list, and Shadowserver counted over 5,000 exposed, unpatched instances in late January 2024.

**How it maps to this topic**

| Password-change flaw (Academy) | GitLab CVE-2023-7028 |
|---|---|
| Client supplies the `username` that decides which account is modified | Client supplies the email list that decides who receives the reset link |
| Server doesn't confirm the request is legitimately tied to that account | Server doesn't confirm the extra address belongs to the account owner |
| Attacker can target arbitrary users | Attacker can target any user whose email they know |
| Root cause: trusting client-controlled input for identity | Root cause: same |

### A second, smaller example: client-controlled `User_Id`

A bug-bounty write-up (private program, company not named) describes an API endpoint for password reset that accepted just two fields: a `User_Id` and a `New_Password`. No authentication, no current password, no token. Changing `User_Id` from the tester's own value to another user's value reset *that* user's password, which the researcher confirmed by logging in as the victim on an authorized test account. It was validated as a critical mass account takeover. Same class of bug as the lab above, just without any of the safeguards (and without even an oracle needed — the server would reset *any* user's password on request).

---

## How to defend against it

- **Never take the target user from the request.** Derive it from the authenticated session on the server. Hidden fields are user-editable, so they're not a security control.
- **Require authentication** for password change endpoints; don't expose them to anonymous users.
- **Apply the same protections as login:** rate limiting, account lockout, and monitoring for repeated failures.
- **Use one generic error message** for "wrong current password" and any other failure, so the response doesn't act as an oracle. Keep response timing and size consistent too.
- **Validate input types strictly.** Reject arrays or unexpected parameters where a single value is expected (the GitLab bug was essentially a list where one address was intended).
- **For reset flows:** send links only to verified addresses, use single-use, short-lived, unpredictable tokens, and re-validate the token when the new password is submitted.
- **Add defense in depth:** require MFA or re-authentication for password changes, email the user when their password changes, and invalidate their other sessions afterward.

---

## Quick testing checklist (authorized targets only)

- [ ] Is the password change reachable **without logging in**?
- [ ] Does the request contain a **username / user ID / email** field (including hidden ones)? What happens if you change it?
- [ ] Do the error messages differ between "wrong current password", "passwords don't match", and "success"?
- [ ] Is there **rate limiting or lockout**, and is it as strict as on the login page?
- [ ] Can you send **extra or duplicate parameters** (arrays, repeated keys) and change who receives a link or whose password changes?
- [ ] Are other sessions logged out after a password change, and is the user notified?

---

## Sources

- PortSwigger Web Security Academy, [Vulnerabilities in other authentication mechanisms](https://portswigger.net/web-security/authentication/other-mechanisms)
- iTnews, [GitLab fixes account takeover vulnerability](https://www.itnews.com.au/news/gitlab-fixes-account-takeover-vulnerability-604086)
- Security Affairs, [GitLab fixed a critical zero-click account hijacking flaw](https://securityaffairs.com/157389/security/gitlab-zero-click-account-hijacking-flaw.html)
- Security Affairs, [5379 GitLab servers vulnerable to zero-click account takeover attacks](https://securityaffairs.com/158075/hacking/gitlab-servers-vulnerable-cve-2023-7028.html)
- stack.watch, [CVE-2023-7028 summary](https://stack.watch/vuln/CVE-2023-7028)
- DEV Community, [$35,000 Bounty: How Inappropriate Access Control Led to GitLab Account Takeover](https://dev.to/cyberw1ng/35000-bounty-how-inappropriate-access-control-led-to-gitlab-account-takeover-3imm)
- InfoSec Write-ups, [I Changed One "User_Id" and the API Said "Sure"](https://infosecwriteups.com/i-changed-one-user-id-and-the-api-said-sure-from-password-reset-to-mass-account-takeover-9d4d4e15e022)
