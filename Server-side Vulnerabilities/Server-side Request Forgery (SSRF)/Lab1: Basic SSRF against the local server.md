# Server-Side Request Forgery (SSRF)

## What It Is

SSRF happens when an attacker can trick a server into making an HTTP request on the attacker's behalf, to a destination the attacker chose rather than the destination the developer intended. The application does the fetching, not the attacker's browser — which is exactly why it's dangerous. The server is usually sitting in a trusted position on the network (inside a VPC, behind a firewall, able to reach internal admin panels or cloud metadata endpoints), and SSRF lets an outsider borrow that position.

The root cause is almost always the same shape: some feature needs to fetch a resource based on user input (a URL, a hostname, an IP, a file path that gets resolved into a fetch), and the backend doesn't restrict *where* that fetch is allowed to go.

## The Loopback Trick, Explained

A classic pattern: imagine an internal tool that lets employees generate a PDF preview of any webpage by submitting a URL — `POST /generate-preview` with a body like `url=https://example.com/page`. The service fetches that URL server-side and renders it.

Normally this looks harmless. But if the server will fetch *any* URL supplied to it, an attacker can submit:

```
url=http://127.0.0.1:8080/admin/users
```

or

```
url=http://localhost:9200/_cluster/health
```

Now the request to `/admin/users` isn't coming from a random IP on the internet — it's coming from the server itself, over its loopback interface. A huge number of internal admin panels, monitoring dashboards, and databases (Elasticsearch, Redis, internal APIs) either skip authentication for local traffic entirely or sit behind a network boundary that only checks *where a request enters the network*, not who is actually asking. The PDF-preview service becomes a proxy that walks straight past both of those defenses.

## Why "Local" Traffic Gets a Free Pass

This trust gap tends to come from a few recurring design decisions, not one single mistake:

- **Access control lives in the wrong layer.** A reverse proxy or gateway enforces auth on the way in, but anything already inside the network — including the app server looping back to itself — is assumed to be pre-vetted and skips that layer entirely.
- **A deliberate "break glass" backdoor.** Some admin tools intentionally allow unauthenticated access from `localhost` so an administrator who's lost their credentials can still recover the system by logging into the box directly. The assumption is that *only* a trusted operator would ever be sitting on that machine — an assumption SSRF quietly breaks.
- **Security through obscure ports.** An admin interface might run on a separate, non-standard port that's never exposed to the internet, so nobody bothered to put real authentication in front of it. SSRF doesn't care what port it's on — the vulnerable server can reach any port on itself.

In cloud environments this exact pattern shows up in an even higher-stakes form: instance metadata services.

## Real-World Example: The Capital One Breach (2019)

The clearest large-scale illustration of SSRF isn't a toy lab — it's the Capital One breach, where a former AWS employee exploited a misconfigured web application firewall to reach AWS's instance metadata service and pull temporary credentials for an over-permissioned IAM role.

Roughly what happened:

1. **The vulnerable component.** Capital One was running a WAF instance that had been assigned an IAM role with far more permissions than it needed.
2. **The SSRF.** The attacker found a way to make that WAF issue requests on her behalf to AWS's EC2 metadata endpoint — the same trick as the `localhost` example above, just aimed at the cloud's built-in "loopback" address, `169.254.169.254`, which every AWS compute instance can query to get information about itself, no special headers required.
3. **Credential theft.** That metadata endpoint handed back the temporary access credentials tied to the WAF's IAM role — a role internally named ISRM-WAF-Role — which happened to include access to S3 storage. Because Capital One was still running the first version of the metadata service (IMDSv1), which answered plain, unauthenticated requests, there was nothing stopping the SSRF from simply asking for the keys and getting them.
4. **The exfiltration.** With valid AWS credentials in hand, the attacker pulled records tied to roughly 106 million credit card applicants across the US and Canada — around 30GB of structured and semi-structured data pulled from S3, much of it unencrypted.
5. **The aftermath.** Capital One was hit with an $80 million penalty from the OCC and later settled a class action for $190 million; the attacker was arrested and later convicted on computer fraud charges.

The fix AWS shipped afterward is a good mental model for SSRF defense in general: IMDSv2 requires a session token fetched via a separate authenticated call before the metadata service will respond, which is specifically designed to make this class of SSRF exploitation much harder. In other words — stop letting a bare, unauthenticated GET request unlock privileged data, even if that GET request is "only" coming from inside the house.

## Mitigations Worth Remembering

- **Never let user input build a request URL unchecked.** Validate against an allowlist of expected hosts/schemes rather than trying to blocklist `127.0.0.1`, `localhost`, `0.0.0.0`, IPv6 loopback, decimal/octal IP encodings, and DNS rebinding — blocklists for this are notoriously leaky.
- **Segment the network so "internal" isn't a synonym for "trusted."** Admin interfaces should require real authentication regardless of where the request originates.
- **Disable or lock down metadata services when they aren't needed**, and prefer versions (like IMDSv2) that require an explicit authentication step.
- **Apply least privilege to any service that fetches external resources on a user's behalf** — a WAF, a PDF generator, a webhook validator, an image proxy — so that even a successful SSRF has nothing valuable to steal.

---

## Lab Walkthrough: Basic SSRF Against the Local Server

**Source:** PortSwigger Web Security Academy
**Vulnerability class:** SSRF (loopback trust bypass)
**Goal:** Delete the user `carlos` by reaching admin functionality that the app only exposes to requests coming from its own server.

### Setup

This lab has a stock-check feature that fetches data from an internal system — the frontend sends a URL, and the backend goes and fetches whatever URL it's given. That's the whole vulnerability in one sentence: the server will retrieve any URL handed to it, including ones pointing back at itself.

### Steps

**1. Trigger the vulnerable request.**
With Burp Suite's intercept turned on, I opened a random product, clicked into its details page, scrolled down, and hit **Check stock**.

![check stock button](Images/1check_stock_button)

**2. Capture the request.**
The intercepted request carries a `stockApi` parameter pointing at the internal stock API, something like:

![stockAPI request](Images/1stockAPI_before_changing_request)

**3. Redirect the fetch to the local admin panel.**
I replaced the `stockApi` value with `http://localhost/admin`:

![stockAPI request after adding //localhost/admin](Images/1stockAPI_after_changing_request)

The reason this works ties straight back to the loopback trick above: the server does the fetching, not my browser. Once the `stockApi` value points at `localhost`, the request to `/admin` originates from the app server itself. The admin panel doesn't ask for credentials because it was built on the assumption that only the server (or someone standing at it) could ever reach that address — it never anticipated being asked to fetch that address *on someone else's behalf*.

**4. Forward the request and land on the admin panel.**
Forwarding it through Burp returns the admin interface — no login required, because the request is (as far as the app can tell) local, trusted traffic:

![admin access](Images/1admin_access)

**5. Delete `carlos`.**
From inside the now-accessible admin panel, I deleted the user `carlos`, completing the lab.

### Takeaway

This lab is the loopback pattern in its purest form: one parameter (`stockApi`) with zero destination validation, an admin panel that trusts anything arriving over `127.0.0.1`/`localhost`, and a feature that's happy to be turned into a proxy. It's the same mechanism as the Capital One case above, just without a cloud metadata service in the middle — the fix in both cases is identical: never let the destination of a server-side fetch be fully attacker-controlled, and never let "the request came from localhost" substitute for actual authentication.
