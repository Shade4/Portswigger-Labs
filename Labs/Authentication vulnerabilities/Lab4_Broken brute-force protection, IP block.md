# Flawed Brute-Force Protection — Concept & Lab Walkthrough

**Lab:** [Broken brute-force protection, IP block](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)
**Category:** Authentication — Password-based logic flaws
**Tooling:** Burp Suite (Proxy, Intruder — Pitchfork attack)

---

# Part I — The Concept: Flawed Brute-Force Protection

## The idea

A login form is one of the few places an attacker gets unlimited guesses unless something stops them. Apps usually stop them with one of two counters:

- **Account lockout** — lock the *account* itself after N wrong passwords, regardless of where the requests are coming from.
- **IP-based blocking / rate limiting** — lock the *source* instead, cutting off an IP address once it's racked up too many failures in a short window.

Neither approach makes the password harder to guess. They just make guessing slow enough, or disruptive enough, that automating it stops being worth it — *if* the counter logic actually holds.

## Where it breaks

The whole defense lives or dies on one question: **when does the counter reset?**

A common but flawed answer is "reset it the moment a login succeeds" — the assumption being that a successful login proves the requester isn't an attacker. That assumption only holds if the *thing that reset the counter* and the *thing being protected* are the same principal. When the reset is scoped too broadly — e.g. an IP-based counter that resets on *any* successful login from that IP, not just the account being targeted — the logic can be trivially gamed by anyone who happens to hold one valid credential on the same system: their own.

The attacker's playbook is then just:

1. Fire off a batch of failed guesses against the victim's account — enough to get close to the lockout threshold, never over it.
2. Log in once with their *own* valid credentials, from the same IP.
3. Watch the failure counter reset to zero.
4. Repeat for the rest of the wordlist.

In Burp Intruder terms, that's as simple as seeding your own working credentials into the payload list at a fixed interval (every *N*-1 attempts, where *N* is the lockout threshold) so the count never reaches the limit.

The fix isn't to stop resetting the counter — that just turns the feature into a denial-of-service against real users after one bad password day. The fix is scoping the reset correctly: tie it to the account that actually authenticated, not to everything sharing its IP or session.

## Real-world example

**F5 BIG-IP ASM — Bug ID 641559** (opened 2017, fixed in 13.1.0) is close to a textbook case of exactly this flaw. ASM's session-based brute-force protection tracked failed login attempts per browser session/cookie and blocked further attempts once failures crossed a threshold (default: 5). The bug: if the end user logged in *successfully* before hitting that threshold, the failed-attempt counter reset to zero — letting an attacker who occasionally supplies a correct credential rack up unlimited failed guesses in between, never tripping the lockout. F5's own advisory lists the workaround as simply "None" until the fix shipped.

A more recent variant, **CVE-2026-44195** in OPNsense (fixed in 26.1.7), shows the same root cause showing up in a different shape: a logic flaw in OPNsense's `lockout_handler` let an unauthenticated attacker keep resetting their own IP's authentication failure counter by submitting login attempts with a crafted *username* containing a success-like keyword (e.g. "Accepted" or "Successful login"). The handler's reset logic trusted a signal that wasn't actually verifying a real successful authentication, so the attacker could brute-force indefinitely without ever reaching the lockout threshold — tracked under CWE-307, Improper Restriction of Excessive Authentication Attempts.

Both cases boil down to the same design mistake as the lab below: the system trusted a "success" signal it hadn't properly bound to the specific login attempt it was supposed to protect.

## Takeaway

Lockout and rate-limiting logic needs to answer, precisely:

- Reset scoped to *what* — the specific account, not the IP/session as a whole?
- Triggered by *whose* success — verified, authenticated success tied to the account under attack, not any adjacent signal that merely looks like success?

Get either of those wrong, and the protection is bypassable by anyone who can produce one legitimate "success" event on demand. The rest of this document walks through exactly that, against a real (if deliberately vulnerable) login form.

---

# Part II — Lab Walkthrough: Broken Brute-Force Protection, IP Block

## Objective

The lab presents a login form protected by an IP-based lockout: submit enough wrong passwords in a row and your IP is temporarily blocked. The goal is to recover the password for the victim account `carlos` from a supplied wordlist, then log in and reach `/my-account`, by putting the Part I concept into practice.

**Given credentials:** `wiener:peter` (your own, always-valid account)
**Target account:** `carlos`
**Candidate passwords:** a ~100-entry wordlist of common passwords (included in full below)

### The candidate password wordlist used for `carlos`

```
123456, password, 12345678, qwerty, 123456789, 12345, 1234, 111111, 1234567, dragon,
123123, baseball, abc123, football, monkey, letmein, shadow, master, 666666, qwertyuiop,
123321, mustang, 1234567890, michael, 654321, superman, 1qaz2wsx, 7777777, 121212, 000000,
qazwsx, 123qwe, killer, trustno1, jordan, jennifer, zxcvbnm, asdfgh, hunter, buster,
soccer, harley, batman, andrew, tigger, sunshine, iloveyou, 2000, charlie, robert,
thomas, hockey, ranger, daniel, starwars, klaster, 112233, george, computer, michelle,
jessica, pepper, 1111, zxcvbn, 555555, 11111111, 131313, freedom, 777777, pass,
maggie, 159753, aaaaaa, ginger, princess, joshua, cheese, amanda, summer, love,
ashley, nicole, chelsea, biteme, matthew, access, yankees, 987654321, dallas, austin,
thunder, taylor, matrix, mobilemail, mom, monitor, monitoring, montana, moon, moscow
```

## 1. Reconnaissance: Observing the Lockout Behaviour

With Burp running as the intercepting proxy, open the lab, go to **My account**, and submit an obviously wrong login (e.g. `test:test`) a few times in a row.

Two things become observable:

1. Submitting a valid username with the wrong password returns a response indicating an incorrect password — distinct from a generic "invalid credentials" message, which quietly confirms `carlos` is a real account (the target username is already given in this lab, so this is just a confirmation, not an enumeration step).
2. After a small number of consecutive failures — **3**, per the lab's own documentation — the app stops evaluating credentials altogether and instead returns a message to the effect of *"you have made too many incorrect login attempts, try again in 1 minute(s)"* — the IP-block kicking in.

Switch to **Proxy > HTTP history** in Burp and locate the `POST /login` request generated by one of these attempts. This is the request you'll pivot into Intruder.

![HTTP history POST login request captured](Images/Lab%204/POST_request_login.png)

A representative version of that request looks like this:

```http
POST /login HTTP/2
Host: <lab-id>.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 29

username=carlos&password=test
```

The three response categories you're working with throughout this lab are:

| Scenario | Status | Body (paraphrased) |
|---|---|---|
| Wrong password for a real account | `200 OK` | Message indicating the password is incorrect |
| 3 consecutive failures reached | `200 OK` | Message stating too many attempts have been made; try again shortly |
| Correct credentials | `302 Found` | Redirect to `/my-account`, with a new session cookie set |

The `302` is the signal we're ultimately hunting for — it's the only response type that indicates a successful authentication, whether that's `wiener`'s known-good login or the one lucky guess against `carlos`.

## 2. Building the Pitchfork Attack

Right-click the captured `POST /login` request and choose **Send to Intruder**.

1. **Positions tab** — set the attack type to **Pitchfork**. This attack type walks two (or more) payload lists *in lockstep*, position by position, rather than trying every combination (Sniper) or every pairing (Cluster bomb/Battering ram). That's exactly what's needed here: request 1 must use username-list[0] with password-list[0], request 2 must use username-list[1] with password-list[1], and so on, in a fixed, predictable order.
2. Click **Clear §** to strip Burp's automatically-guessed payload positions.
3. Highlight the value of the `username` parameter and click **Add §** to mark it as payload position 1. Do the same for the `password` parameter as payload position 2.

At this point the request template looks like:

```http
POST /login HTTP/2
Host: <lab-id>.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: ...

username=§carlos§&password=§test§
```

### Why concurrency has to be forced down to 1

Go to the **Resource pool** tab, create (or assign) a pool for this attack, and set **Maximum concurrent requests** to `1`.

This step is easy to skip and is the single most common reason people fail to reproduce this attack. Burp Intruder normally fires several requests in parallel for speed. If two or more requests are in flight simultaneously, the server can receive them out of order — for example, a failed `carlos` guess arriving *after* the `wiener` reset that was supposed to precede it, rather than before it. Because the entire attack depends on a strict alternating sequence (guess → reset → guess → reset...), any reordering risks stacking up three real failures in a row and triggering the IP block mid-attack, corrupting the results. Capping concurrency at 1 forces Burp to send requests strictly one at a time, in list order, preserving the sequence the attack depends on — this is exactly the "reset scoped to what, triggered by whose success" logic from Part I, being exploited on purpose.

## 3. Constructing the Interleaved Payload Lists

The core exploitation technique, straight out of Part I's attacker playbook: **interleave your own valid login into both payload lists so it lands before the 3-failure threshold is ever reached.** With a 3-strikes lockout, alternating every single request is a comfortable safety margin — it guarantees at most one failure ever accumulates before the counter is reset back to zero.

So instead of two plain wordlists, both the username and password payload lists become an alternating sequence:

```
username list          password list
------------------      ------------------
carlos                  <candidate #1>
wiener                  peter
carlos                  <candidate #2>
wiener                  peter
carlos                  <candidate #3>
wiener                  peter
...                     ...
```

Building this by hand for a ~100-entry wordlist is tedious and error-prone, so it's worth generating both lists programmatically from the raw candidate-password wordlist:

```python
# build_payloads.py
# Reads the raw candidate password wordlist and produces the two
# Pitchfork payload lists, interleaving a known-good login (wiener:peter)
# after every guess so the IP-block counter never reaches its threshold.

with open("candidate_passwords.txt") as f:
    candidates = [line.strip() for line in f if line.strip()]

usernames, passwords = [], []
for pwd in candidates:
    usernames.append("carlos")
    passwords.append(pwd)
    usernames.append("wiener")
    passwords.append("peter")

with open("usernames.txt", "w") as f:
    f.write("\n".join(usernames))

with open("passwords.txt", "w") as f:
    f.write("\n".join(passwords))
```

Running this produces two files whose first several lines look like:

```
usernames.txt        passwords.txt
--------------        --------------
carlos                123456
wiener                peter
carlos                password
wiener                peter
carlos                12345678
wiener                peter
carlos                qwerty
wiener                peter
```

...continuing in that pattern for the full wordlist. Paste `usernames.txt` into the **Payload set 1** list and `passwords.txt` into the **Payload set 2** list in Intruder's Payloads tab, then click **Start attack**.

## 4. Running the Attack and Isolating the Correct Password

Once the attack completes, every row falls into one of the three response categories from the table in Step 1. In practice:

- Every `wiener` row returns `302 Found` — its credentials are always correct, so it always succeeds and always resets the counter.
- Almost every `carlos` row returns `200 OK` — a failed guess.
- **Exactly one** `carlos` row returns `302 Found` — the correct password.

Since both the routine `wiener` resets and the one correct `carlos` guess return the same `302` status, the status code alone doesn't isolate the answer — it just narrows the haystack. Click the **Status code** column header to sort, or use the results-table filter to hide the common `2xx` "no news" bucket:

![view filter](Images/Lab%204/view_filter_2xx.png)

With the noise filtered out, scan the remaining `302` rows and match the one where the **username** column reads `carlos` rather than `wiener`. That row's password payload is the answer.

![Credentials found](Images/Lab%204/credential_found_for_carlos.png)

## 5. Confirming the Solve

Go back to the login page, submit `carlos` with the recovered password, and load `/my-account`. If the block is still cooling down from an earlier manual test, waiting the stated 1 minute clears it. Once the account page loads for `carlos`, the lab is marked as solved.

## 6. Tying It Back to Part I

This lab is a direct, hands-on instance of the "Where it breaks" section above: the IP-based counter resets on *any* successful login from that IP — including `wiener`'s, which has nothing to do with the account actually under attack (`carlos`). Because the reset isn't scoped to the account being protected, holding one other valid credential on the same system (`wiener:peter`) is enough to defeat the entire mechanism, exactly as described in the general attacker playbook, with the lab's specific threshold at 3 consecutive failures.

---

## References

- [PortSwigger Web Security Academy — Broken brute-force protection, IP block](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)
- [F5 Bug ID 641559 — Session-based brute force resets failed logins counter upon successful login](https://cdn.f5.com/product/bugtracker/ID641559.html)
- [CVE-2026-44195 — OPNsense authentication lockout bypass](https://fieldguide.lutrasecurity.com/CVE-2026-44195/)
