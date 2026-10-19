NIP-XX
======

My Other Npubs
--------------

`draft` `optional` `kind:10092` `replaceable`

## Abstract

A replaceable event listing the other Nostr keys a user controls.

Only keys. No usernames, no domains, no accounts on other platforms — those are
[NIP-39](../nips/39.md)'s job, and this event does not repeat it.

## Event

    ["me", "<pubkey>", "<relay>"]

Index 1 is a pubkey the author also controls, required. Index 2 is a relay hint
for that key, optional. Both permutations:

    ["me", "3b7fc823..."]                             another key I control
    ["me", "3b7fc823...", "wss://relay.example.com"]  and where it publishes

In full:

```json
{
  "kind": 10092,
  "content": "",
  "tags": [
    ["me", "3b7fc8231bd4f0e2b1187c3a5e94d0c6a2f8b13e5d7c09a4e6b2f8d1c3a5e7b9", "wss://relay.example.com"],
    ["me", "a1c9e40d7f2b8c5e3a9d1f4b6c8e0a2d5f7b9c1e3a5d7f9b1c3e5a7d9f1b3c5e"]
  ]
}
```

`content` is unused. Skip any `me` tag whose pubkey is not 64 lowercase hex
characters, and keep the rest of the list. The author's own pubkey does not
belong here — ignore it if it appears.

There is deliberately no label element. A name for each key already exists in
that key's own `kind:0`, written by the person who owns it, and a label written
by the lister instead would disagree with it, arrive in one language, and
categorise an account the author may have separated on purpose.

### Withdrawing

The list is full state, not a set of edits: each event replaces the last, so
dropping one key means republishing the others. To clear it entirely, publish an
event carrying nothing:

```json
{ "kind": 10092, "content": "", "tags": [] }
```

Either side can sever a link alone, which is what matters when a key is lost or
stolen — the remaining key does not need cooperation from the one it no longer
controls. The severing is not instant, though. A relay that never receives the
replacement keeps serving the old event, so the link stays visible to anyone
reading from that relay until it does. A [NIP-09](../nips/09.md) deletion request
may go alongside, with the same limitation.

## Confirmation

A `me` tag proves nothing by itself. Anyone can list anyone, and a signature over
the list only shows that the author wrote it.

A client that wants a firmer signal can look for the other half: fetch the named
key's own event and see whether it names the author back. Both events are signed,
so a matching pair takes control of both keys to produce. It costs one extra
fetch per key, and there is no obligation to do it.

That is worth weighing here more than in most places, because the failure has a
victim rather than just a wrong pixel. An unconfirmed claim rendered as a
confirmed link is how impersonation works: a fresh key names a well-known one and
inherits its reputation in any client that shows the claim at face value. Showing
the claim marked as unverified, or not showing it, both avoid that. Showing it as
a link does not.

Confirmation does not chain. A↔B and B↔C being mutual says nothing about A↔C, and
treating it as though it did would let one compromised key drag in every key its
owner ever named. Presenting a confirmed group together is fine; calling an
unstated pair confirmed is not.

## One primary key, not a mesh

Nothing requires every key to list every other. Naming one key as the primary and
having each alternate list only that one is enough: every link a reader sees is
still a mutual pair, confirmed the same way.

The difference is what it costs to add a key. In a full mesh of five keys, a
sixth means re-signing all six events with six different signers, and a set that
is half updated confirms or fails depending on which pair a reader happens to
check. Through a primary it is two signatures, the new key and the primary, and
there is no partial state to land in.

## Why `me` and not `p`

`p` marks someone an event is addressed to — that is what relays index it for,
and what clients route to notifications. A key named here is the event's subject
rather than its audience, and in this case it is the author's own second key,
which needs no telling.

Indexing it would mean `{"#p": ["<pubkey>"]}` returns every event claiming to be
an alternate of that key — the candidate alternates of its owner, from any client,
as a side effect rather than a decision. [NIP-01](../nips/01.md) spells tag
filters as `#<single-letter>`, so there is no `#me` to query.

Worth being plain about the limit: this is not what keeps two identities apart.
The events are public, and anyone with a relay can index whatever they like. It
only means the query is something built on purpose rather than something the
protocol hands out.

Nothing needed is lost. A reader learns about a key by reading the list that
names it, then fetches that key's own event by author to check the other half.

## Relationship to NIP-39

The two sit side by side. [NIP-39](../nips/39.md)'s `kind:10011` `i` tags link a
key to an account on another platform, proved by a post there. This event links a
key to another Nostr key, checked against that key's own statement.

A future revision of NIP-39 could absorb this as a `nostr:` claim type, and the
substance here would survive the move — the check is the design, the tag it rides
on is not. It is not a drop-in, though: an `i` tag carries a proof value, and
every proof NIP-39 defines is a static pointer such as a gist or tweet id. A
counter-claim has no stable identifier, since it lives in a replaceable event
whose id changes each time it is republished. Absorbing this would mean widening
what NIP-39 accepts as a proof, not just adding a name to the list.

## Implementation notes

* A confirmation is only as fresh as the last fetch of both events. Either side
  can withdraw at any time, so a verdict cached for a session is reasonable and a
  verdict cached forever is a stale claim waiting to be shown.
* Listing a key does not follow it, mute it, or grant it anything. It is a
  statement about authorship and nothing else.
* Keys held deliberately offline cannot take part in confirmation, since the
  check needs an event signed by each side. A cold key can be listed; it just
  reads as unconfirmed until it publishes.

## Privacy

The reason for a second key is usually that it is not connected to the first one.
Publishing that connection is irreversible in practice: the event reaches relays
that keep it, and removing it later removes it only from those that receive the
replacement.

What follows from that:

* A shared IP, relay set, posting rhythm or device is the correlation a separate
  key exists to break. A client that suggests a link from those has done the
  correlating on the user's behalf.
* Account creation and key rotation are the two moments a prompt reaches someone
  before they have decided anything.
* Publishing nothing is a working state rather than an incomplete profile. Two
  identities that never touch are using this correctly by never appearing in it.
