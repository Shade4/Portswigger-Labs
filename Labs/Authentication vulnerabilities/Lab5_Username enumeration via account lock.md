# Account Locking as a Brute-Force Defense (and Why It's Not Enough)

## The Concept

Account locking is a common defense where a site temporarily (or permanently) locks
a user's account after a certain number of failed login attempts — usually
somewhere around 3-5 tries. The idea is simple: if someone is guessing passwords
for `alice`, after the 4th or 5th wrong guess, `alice`'s account gets locked, so
the attacker can't keep guessing.

It also leaks information the same way normal login errors do. If a locked
account gives a different response ("This account is locked" vs. "Invalid
username or password"), an attacker can use that response difference to figure
out which usernames actually exist on the system, even without ever seeing a
password.

The core problem with account locking is that it's built to stop someone
hammering **one specific account** with lots of password guesses. It says
nothing about someone trying **one or two guesses each** against **thousands of
accounts**.

### Bypass #1: Password Spraying

If the lockout threshold is, say, 3 attempts, an attacker doesn't need to guess
3 passwords against one account. Instead they can:

1. Build a list of likely usernames (scraped from LinkedIn, leaked breach data,
   enumerated via the site's own login form, or just `firstname.lastname`
   patterns).
2. Pick 2-3 passwords that are common enough that *someone* on that list is
   likely using one of them (`Summer2024!`, `Password123`, `CompanyName1`).
3. Attempt each of those 2-3 passwords against every username, staying under
   the lockout threshold for every single account.

No account ever sees more than 2-3 failed attempts, so nothing locks — but
across a list of 10,000 usernames, the odds that at least one person is using
one of those weak passwords are extremely high. This is called **password
spraying**, and it flips the brute-force script: instead of many guesses on
one account, it's few guesses across many accounts.

**Real-world example:** Password spraying is one of the most common techniques
behind large-scale account takeovers of enterprise Microsoft 365 and Azure AD
accounts. Attackers routinely spray a small set of seasonal or default-sounding
passwords across huge lists of corporate email addresses, because most
organizations lock an account after a handful of failed attempts on *that
account*, but don't detect a slow trickle of 1-2 failed logins spread across
the entire company directory. Microsoft's own security team has published
multiple advisories specifically warning about password-spray campaigns
against Office 365 tenants for exactly this reason.

### Bypass #2: Credential Stuffing

Credential stuffing is a different attack that account locking also fails to
stop. Here, the attacker isn't guessing — they already have real
`username:password` pairs, harvested from a completely different site's data
breach. The attack relies entirely on password reuse: if 200 million
credentials leaked from Site A, some meaningful percentage of those people
used the *same* email and password on Site B, C, and D too.

Because each username:password pair is only tried **once** against the
target, there are no repeated failed attempts on any single account for a
lockout policy to catch. The attacker isn't brute-forcing in the traditional
sense at all — they're just checking which stolen credentials happen to also
work elsewhere, at massive scale, usually with automated tools.

**Real-world example:** The 2019-2020 wave of Disney+ account takeovers, which
happened within hours of the service's launch, is a textbook credential
stuffing case. Attackers didn't crack Disney's systems at all — they replayed
username/password combinations from older, unrelated breaches (like the
Collection #1 leak) against Disney+'s login page. Anyone who reused a breached
password on Disney+ had their account compromised almost immediately, and
because each pair was only tried once per account, ordinary lockout policies
did nothing to stop it.

### Takeaway

Account locking is still worth having — it does raise the bar for a
targeted attack on one specific person. But on its own it's a false sense of
security against the two most common large-scale credential attacks:

- **Password spraying** gets around it by keeping attempts-per-account low
  and spreading them across many accounts.
- **Credential stuffing** gets around it by never repeating a guess on the
  same account at all.

Real protection needs to layer in things account locking can't provide alone:
rate-limiting by IP/device, CAPTCHAs, multi-factor authentication, and
detecting anomalous login patterns across the whole user base rather than per
account.

---

# Lab Walkthrough: Username Enumeration via Account Lock

**Lab:** PortSwigger Web Security Academy — *Username enumeration via account
lock* (Practitioner)

**Goal:** This lab's lockout mechanism has a logic flaw that leaks whether a
username exists. The task is to enumerate a valid username, brute-force that
user's password, then load their account page.

This lab is the practical demonstration of the "flawed brute-force logic"
problem above: it doesn't demonstrate password spraying or credential
stuffing directly, but it shows the *other* classic weakness in account
locking — the lockout state itself becomes an oracle that reveals valid
usernames, because a locked account's error response is distinguishable from
an "invalid username or password" response.

## Step 1 — Baseline request

With the browser proxied through Burp, submit any username/password on the
lab's login form (e.g. `test` / `test`) to capture the shape of the request,
then send that `POST /login` request to Intruder. A raw request looks like
this:

```
POST /login HTTP/2
Host: 0aXX...web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 29

username=test&password=test
```

![login POST request](Images/Lab%205/sending_POST_request_login_to_intruder.png)

## Step 2 — Cluster bomb: find the username

Two payload positions are marked — one on `username`, one on `password` —
and the attack type is set to **Cluster bomb**. Cluster bomb tries *every*
combination of the two payload sets against each other (every username with
every password), as opposed to **Pitchfork**, which walks both lists in
lockstep (item 1 with item 1, item 2 with item 2, etc.) and never mixes
positions across different indices.

```
username=§invalid-username§&password=§example§
```

The candidate username list (`carlos`, `admin`, `test`, hundreds of common
account names, etc.) goes into the first payload set, and the candidate
password list (`123456`, `password`, `qwerty`, etc.) goes into the second.

Before starting, a **Grep - Extract** rule is added under the attack's
settings: fetch a baseline response and select the literal text
`Invalid username or password.` so Burp adds a column showing whether each
response contains that exact string.

![Grep Extract section](Images/Lab%205/fetching_response_and_inputting_it_Grep_Extract.png)

Running the attack, nearly every response comes back with
`Invalid username or password.` in the Grep - Extract column — except for one
username, whose responses instead show a different message:

```
You have made too many incorrect login attempts. Please try again in X minute(s).
```

That message only appears because *that* username exists and has already
been "locked" by the repeated attempts cluster bomb just threw at it. Any
username that doesn't exist can never lock, so it always falls back to the
generic invalid-credentials message. That distinction is the whole
vulnerability: **the site tells you which accounts are real by how they fail
to log in.** Note that username and stop the attack — continuing wastes
requests and keeps the account locked longer.

![username found](Images/Lab%205/finding_username_through_response.png)

## Step 3 — Sniper: brute-force the password

Start a new Intruder attack on the same `POST /login` request, this time
with attack type **Sniper**. Sniper only ever moves one payload position at a
time, which is exactly what's needed now: the username is fixed to the one
identified in Step 2, and only the `password` field is marked as a payload
position:

```
username=carlos&password=§example§
```

The password candidate list is loaded into that single position, along with
the same Grep - Extract rule for the error string. After waiting long enough
for the account's lock to expire (the lab enforces a cooldown window), run
the attack.

Scanning the Grep - Extract column, most responses still show
`Invalid username or password.` — but one response has an **empty** Grep -
Extract cell, meaning that response didn't contain the error text at all.
That's the correct password: a successful login doesn't return the failure
message, so its absence is the tell.

![password found](Images/Lab%205/sniper_attack_password_found.png)

## Step 4 — Log in and confirm

Enter the identified `username:password` pair into the actual login form
(waiting out any remaining lockout cooldown first if needed), log in, and
open the account page. Lab solved.

### Alternative: single-pass cluster bomb

Steps 2-3 above split the work into two focused attacks to keep things fast
and easy to read in the results table. But it's not strictly necessary to
split them: if the original **Cluster bomb** attack from Step 2 is just left
to run to completion (every username × every password), it will eventually
land on the one combination that returns a successful login — the username
and the password both fall out of that single attack, with no need to start
a second Sniper attack afterwards.

The trade-off is volume and time: cluster bomb sends `len(usernames) ×
len(passwords)` requests, so letting it run start-to-finish is far noisier
and slower than stopping early once the locked username is spotted. It's
also more likely to trip additional protections (rate limiting, WAFs) in a
real target, since it's hammering every account on the list with every
password rather than stopping once the target is identified. Splitting into
two attacks is the more surgical, realistic approach; running cluster bomb
to completion is the "brute force everything and let the results speak for
themselves" approach — both get you the same answer.

![cluster bomb one go](Images/Lab%205/cluster_bomb_attack_one_go.png)

### Official PortSwigger solution (for reference)

> 1. With Burp running, investigate the login page and submit an invalid
>    username and password. Send the `POST /login` request to Burp Intruder.
> 2. Select Cluster bomb attack from the attack type drop-down menu. Add a
>    payload position to the username parameter. Add a blank payload position
>    to the end of the request body by clicking **Add §**. The result should
>    look something like this: `username=§invalid-username§&password=example§§`
> 3. In the Payloads side panel, add the list of usernames for the first
>    payload position. For the second payload position, select the **Null
>    payloads** type and choose the option to generate 5 payloads. This will
>    effectively cause each username to be repeated 5 times. Start the attack.
> 4. In the results, notice that the responses for one of the usernames were
>    longer than responses when using other usernames. Study the response
>    more closely and notice that it contains a different error message:
>    *You have made too many incorrect login attempts.* Make a note of this
>    username.
> 5. Create a new Burp Intruder attack on the `POST /login` request, but this
>    time select **Sniper** attack from the attack type drop-down menu. Set
>    the username parameter to the username that you just identified and add
>    a payload position to the password parameter.
> 6. Add the list of passwords to the payload set and create a grep
>    extraction rule for the error message. Start the attack.
> 7. In the results, look at the grep extract column. Notice that there are a
>    couple of different error messages, but one of the responses did not
>    contain any error message. Make a note of this password.
> 8. Wait for a minute to allow the account lock to reset. Log in using the
>    username and password that you identified and access the user account
>    page to solve the lab.

The official solution is a slight variant of Step 2 above: instead of pairing
each username with the *same* password list used later for brute-forcing, it
repeats each username 5 times against **Null payloads** (blank second
position) purely to trigger the lockout state and inflate the response length
for the real account — a cleaner way to surface the lockout message without
mixing it into the eventual password-guessing payloads. Functionally, it
reaches the same conclusion as the cluster bomb approach used above.

### Why this matters beyond the lab

This lab is a narrow, single-account version of the enumeration problem, but
it's the same failure mode that makes password spraying and credential
stuffing effective in the real world: **any observable difference in server
behavior based on whether a username is valid** — a distinct error string, a
different HTTP status code, a timing difference, or, as here, a distinct
lockout message — hands an attacker a free username oracle. Combine that
oracle with the spraying technique described above (few passwords, many
usernames, always staying under the lock threshold) and an attacker no
longer needs the lab's cooldown-and-retry dance at all; they just never trip
any single account's lock in the first place.
