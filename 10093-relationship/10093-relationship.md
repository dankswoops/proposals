NIP-XX
======

Relationship Status
-------------------

`draft` `optional` `kind:10093` `replaceable`

## Abstract

A replaceable event carrying one code for a user's relationship status, and
optionally the pubkey of the person it is with.

One code, one person at most. It is not a relationship graph and not a presence
signal.

## Values

| code | meaning |
|------|---------|
| `0` | single |
| `1` | in a relationship |
| `2` | engaged |
| `3` | married |
| `4` | in a civil partnership |
| `5` | separated |
| `6` | divorced |
| `7` | widowed |
| `8` | it's complicated |

The meanings are a reference for translators, not the wire format. The value is a
key: a client shows its own wording for a code it knows and nothing for a code it
does not, which is also how the list grows without breaking older readers.

**Codes are permanent.** Once assigned, a code keeps its meaning forever and
anything new takes the next free number. Renumbering would silently change what
already-published events say.

Civil union, domestic partnership and common-law all map to `4` — the same
arrangement under different paperwork, which no client renders differently.
"Prefer not to say" needs no code: publish nothing, or withdraw.

Numbers rather than the string enums of [NIP-52](../nips/52.md) and
[NIP-53](../nips/53.md), because those are machine states (`accepted`, `live`)
that clients rarely translate, while these are user-facing nouns with no neutral
rendering across locales. Shipping English on the wire would push a translation
problem into every client.

## Event

    ["status", "<code>", "<pubkey>", "<relay>"]

Index 1 is the code and is required. Index 2 is the pubkey of the person the
status is with, optional. Index 3 is a relay hint for that pubkey, optional.
Every permutation:

    ["status", "0"]                                            single
    ["status", "6"]                                            divorced
    ["status", "3", "97c70a44..."]                             married to them
    ["status", "3", "97c70a44...", "wss://relay.example.com"]  married to them, and where to find them

In full:

```json
{
  "kind": 10093,
  "content": "",
  "tags": [
    ["status", "3", "97c70a44366a6535c145b333f973ea86dfdc2d7a99da618c40c64705ad98e322", "wss://relay.example.com"]
  ]
}
```

`content` is unused. Read the first `status` tag and ignore any others. Skip a
code that is missing, non-integer or outside the list — render nothing rather
than guessing. Skip index 2 unless it is 64 lowercase hex characters; the code
still stands on its own.

A code naming nobody is complete, not a degraded form. `6` and `7` will almost
always appear alone, and `0` can never name anyone.

### Withdrawing

A replaceable event cannot be unsent, so clearing the field means publishing a
newer one that carries nothing:

```json
{ "kind": 10093, "content": "", "tags": [] }
```

The newest event wins, so this overwrites the old value wherever it reached. A
[NIP-09](../nips/09.md) deletion request may go alongside, but it reaches only
the relays that receive it, whereas the replacement travels the same path the
original did. For a field like this one, removal often matters more than
publication, so it is worth making a single obvious action in the UI.

A reader has two outcomes: a code, or nothing. Whether the silence is "never
published" or "published, then withdrawn" is invisible, and does not need to be
visible.

## Why the pubkey is not in a `p` tag

`p` marks someone an event is addressed to — that is what relays index it for,
and what clients route to notifications. The person named here is the event's
subject rather than its audience: it describes them, but nothing in it is written
to them.

[NIP-01](../nips/01.md) defines `p` as referring to another user, and indexing one
here would let `{"#p": [...]}` answer "who claims a relationship with this person"
for anyone who asks — incidentally, from any client, without anyone deciding it
should be possible. A multi-letter tag is not a filter key, since NIP-01 spells
tag filters as `#<single-letter>`.

That is a default, not a protection. The events are public and signed, so the
same list is a short script away for anyone with a relay or a firehose. The
choice only decides whether it takes that decision. The shape follows [NIP-57](../nips/57.md) Appendix G,
`["zap", "<pubkey>", "<relay>", "<weight>"]`, with the code moved to the front
because it is the one element always present.

The cost is that the named person is not notified. They find out when somebody
tells them, which for this field is the more considerate default.

## Confirmation

A status naming someone is a claim, and claims are one-sided — anyone can name
anyone. The useful signal is agreement: fetch the named pubkey's own `kind:10093`
and see whether it names the author back with the same code. That is a lookup by
author, so it needs no index and nothing is lost by omitting `p`.

Codes that differ are a disagreement rather than a confirmation, and a client is
better off showing neither than picking a winner. Only agreed pairs are worth
drawing as a link between two profiles; an unconfirmed claim reads better as
nothing at all.

## Why not NIP-32 labels

[NIP-32](../nips/32.md) already labels pubkeys, and `kind:1985` with an `L`
namespace would carry roughly this information. Two differences decide it.

`1985` is a regular event, so a list assembled from labels accumulates: removing
one entry means a deletion request rather than replacing the whole statement, and
a reader has to reconcile every label event ever published to know the current
answer. This event is replaceable, so the newest one is the answer.

Label values are free text. `l` of `married` or `casado` renders as whatever the
author typed, to every reader in every language. A fixed set of codes is the only
part of this design a translation layer can work with.

## Implementation notes

* Look for the tag before testing its value. `Number(tag[1]) || fallback` turns a
  legitimate `0` into a miss in JavaScript and Python, and the string `"0"` is
  falsy in Perl and PHP. Presence first, parse second.
* Exclusivity is not expressible. An open relationship publishes `1` and says the
  rest in `about`. Carrying it properly would mean another element for a property
  most users never set.
* One person at most, deliberately. A later revision could permit more by allowing
  further `status` tags: old clients would read the first and show one partner,
  which is incomplete but never wrong. Declaring extra tags invalid today would
  close that door.
* Publishing is a deliberate act. It does not belong in a signup flow or in a
  profile-completeness meter.

## Privacy

Naming someone publishes information about **them**, not only about the author,
and they cannot withdraw it because they do not hold the key that wrote it. Worth
saying at the moment of naming rather than afterwards.

The code alone is a complete statement, and the safer one. Wherever a particular
arrangement carries a cost — legal, social or physical — saying what without
saying who is the whole reason the pubkey is optional.
