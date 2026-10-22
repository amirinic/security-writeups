# BOLA in a Live-Streaming Platform's Developer API — Any User Could Read Any Channel's Subscriber List and Donation Ledger

| | |
|---|---|
| **Program** | Private bug bounty engagement (target anonymized — no public disclosure permission obtained) |
| **Status** | Reported, triaged, and fixed by the vendor |
| **Class** | Broken Object-Level Authorization (BOLA/IDOR) — CWE-639 |
| **Severity** | Medium–High — `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` (6.5) |

## Summary

The platform exposed a read-only "Developer API" that let any registered user generate a free API token in one click. That token authenticated the caller, but the API never checked whether the caller had any relationship — ownership, moderation, or subscription — to the specific channel they queried. Supplying any channel's identifier returned that channel's full subscriber roster, donation ledger, and moderator list, regardless of who was asking.

I confirmed this with a token issued to an ordinary test account that did not own, moderate, or subscribe to the target channel, and read another channel's private data without error.

## Root Cause

The API answered "is this a valid client?" (authentication) but never answered "is this client allowed to see *this* object?" (authorization). The channel identifier was accepted as a trusted lookup key with no ownership/membership check — a textbook BOLA: a user-supplied object reference reaches a data store with no per-object access control gate in between.

## Impact

With one free account and its self-service developer token, an attacker could, for *every* channel on the platform:

- Pull the full **donation ledger**: donor identity, exact amount, message, and timestamp for every transaction.
- Pull the full **subscriber roster**: identity, tier, and start/expiry dates.
- Pull the **moderator roster**.
- Do this in bulk — channel identifiers were sequential/enumerable, so an attacker could walk the entire platform and build a complete "who funds whom / who subscribes to whom" graph, and profile individual donors by spend.


This was read-only — no data could be modified — but on a platform where this data isn't otherwise available in bulk through the UI, and where channel owners had no way to opt out, it amounted to a full breach of subscriber and payment-association privacy, scoped only by attacker patience.

## Why It Wasn't "Working as Intended"

- The data was **not available in bulk anywhere else** — the UI surfaces transient badges/messages, not a queryable, timestamped, exportable roster or ledger.
- Channel owners had **no opt-out** — there was no setting to stop their subscriber/donation data being read this way.
- The access was **fully cross-tenant** — a token minted for account A returned account B's private data with no relationship check at all.


## Proof of Concept (sanitized)

```
GET /api/developer.php?action=channel_tele&username=<ANY_CHANNEL>&limit=50
X-TL-API-Token: <token belonging to an unrelated test account>
```

Response: `success: true`, full page of donation records (`user_id, username, amount, message, created_at`) for a channel the token's owner had no relationship to. The same pattern applied to `channel_subscribers` and `channel_moderators` with the same token and parameter.

| Action | Result | Records returned | Fields |
|---|---|---|---|
| `channel_subscribers` | success | 14 | user_id, username, display_name, tier, months, started_at, expires_at, subscription_id |
| `channel_tele` (donations) | success | 582 | user_id, username, amount, message, created_at, tele_id |
| `channel_moderators` | success | 46 | user_id, username, display_name, moderator_since |

(Real usernames, amounts, messages, and the live token are withheld here — this repo is a technical case study, not a disclosure of the original program's data.)

## Fix

The vendor added a per-channel ownership/relationship check on every `channel_*` action, so a token can only retrieve data for channels its own account owns (or data the owner has explicitly made public). Verified fixed post-remediation.

## Takeaway

Authentication ("do you have a valid token?") and authorization ("are you allowed to touch *this* object?") are two different checks. Any endpoint that takes a user-, channel-, or resource-identifier as a parameter needs an explicit ownership/relationship check before returning data — never trust the identifier alone just because the caller is authenticated.
