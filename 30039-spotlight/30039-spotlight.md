NIP-XX
======

Spotlight
---------

`draft` `optional` `kind:30039` `addressable`

## Abstract

An addressable set naming the people a user wants to see for one category of
content, and the event kinds that category arrives as.

[NIP-51](../nips/51.md) already has both halves of this. Its `kind:30007` kind
mute sets are per-kind lists of pubkeys — *"mute pubkeys by kinds"* — and this is
that with the sign flipped, from exclusion to selection. Its `kind:10020` media
follows is the closest positive form, but with one bucket named by the
specification and applied to everyone in it. Here the bucket is the user's, and
it says which kinds it means.

Spotlighting is not a following (kind-3), a follow list (kind-30000), or a bookmark list (10003/30003). A user can spotlight someone they do not follow, and follow someone they never spotlight.

## Event

A set per category, each identified by its `d` tag:

* `d` — the category name, chosen by the client.
* `k` — repeated, an event kind this category arrives as.
* `star` — repeated, a pubkey to watch for those kinds.

```json
{
  "kind": 30039,
  "content": "",
  "tags": [
    ["d", "reels"],
    ["k", "34236"],
    ["k", "22"],
    ["star", "3b7fc8231bd4f0e2b1187c3a5e94d0c6a2f8b13e5d7c09a4e6b2f8d1c3a5e7b9"],
    ["star", "f0e2b1187c3a5e94d0c6a2f8b13e5d7c09a4e6b2f8d1c3a5e7b93b7fc8231bd4"]
  ]
}
```

That set reads: show me short video from these two people, whether it arrives as
`34236` or as `22`.

The set carries its own kinds, so it is self-describing. A client that has never
heard of the word `reels` still resolves the set correctly, because the meaning
is in the `k` tags rather than in a vocabulary both sides have to share. Category
names need no registry and clients that pick the same ones simply agree.

Skip a `star` that is not 64 lowercase hex and a `k` that is not an integer,
and keep the rest of the set. A set with no `k` tags selects nothing; a set with
no `star` tags watches nobody. Either is a valid empty state.

### A worked set of sets

Ten categories, three people:

    ["d","notes"]     k=[1,6,16]       p=[bob]
    ["d","replies"]   k=[1,1111]       p=[bob]
    ["d","reels"]     k=[34236,22]     p=[alice, carol]
    ["d","videos"]    k=[34235,21]     p=[alice]
    ["d","lives"]     k=[30311]        p=[carol]
    ["d","music"]     k=[32123]        p=[alice]
    ["d","podcasts"]  k=[54]           p=[carol]
    ["d","blogs"]     k=[30023]        p=[bob]
    ["d","calendar"]  k=[31922,31923]  p=[carol]
    ["d","badges"]    k=[8,30009]      p=[carol]

The kinds above are illustrative — whatever a given client watches — not a
registry this document defines.

Two things fall out of the shape. A category usually watches more kinds than any
one client publishes: a client that posts short video as `34236` still lists `22`
alongside it, so an author posting from a different app lands in the feed anyway.
And when a new kind appears for a category, it is one `k` tag added to one set,
covering every author already in it, rather than an edit repeated per author.

### Private entries

The list says what a user wants to read, which is a sharper signal than who they
follow, and it names people who never agreed to being categorised.

Entries can go in either half of the event, following the split
[NIP-51](../nips/51.md) defines: public ones in `tags`, private ones as that same
array stringified and NIP-44-encrypted to the author's own key, stored in
`content`. A reader who is not the author sees only the public half.

Publishing at all is a sync mechanism — the sets have to reach the user's other
devices — rather than a statement addressed to anyone, so the private half is the
better default. A plaintext entry cannot be made private afterwards, because the
copies relays already hold do not change when a newer event says something else.

### Withdrawing a `d` set

Each set is replaced by its own newer version, so removing one person from one
category republishes that set alone. To empty a category, publish it with its `d`
tag and nothing else:

```json
{ "kind": 30039, "content": "", "tags": [["d", "reels"]] }
```

The set now selects nothing, which is the withdrawal. It still exists as an
event, because a replacement is the only thing that travels the same path the
original did.

### Withdrawing the whole thing

There is no single event to clear. [NIP-01](../nips/01.md) addresses these by
`kind`, `pubkey` **and `d` value**, so each category is its own independent slot
and emptying one says nothing about the others.

This is the trap worth naming:

```json
{ "kind": 30039, "content": "", "tags": [] }
```

That does not retire anything. With no `d` tag it addresses the coordinate whose
`d` is the empty string, so it publishes one more set — an unnamed, empty one —
while every named set carries on exactly as before. The same event on a
non-addressable kind would be a withdrawal; here it is an addition.

Clearing everything means emptying each set that was published, one event each. A
[NIP-09](../nips/09.md) request can accompany them, using an `a` tag per
coordinate, since relays are asked to drop all versions up to the request's
`created_at`:

```json
{
  "kind": 5,
  "tags": [
    ["a", "30039:<pubkey>:reels"],
    ["a", "30039:<pubkey>:notes"],
    ["k", "30039"]
  ]
}
```

As always it reaches only the relays that receive it, so the replacements matter
more than the request.

Rewriting one set rather than one list is the reason for this shape. A single
event holding every category grows with authors times categories, is republished
in full every time one star is toggled, and meets relay event size limits at a
few hundred entries — the difficulty `kind:3` contact lists are known for. A set
is rewritten only when its own category changes.

## Reading a spotlight feed

Fetch the sets in one filter:

```json
{ "kinds": [30039], "authors": ["<the reader>"] }
```

Each set then becomes exactly one filter — `k` tags to `kinds`, `star` tags to
`authors`, with nothing to compute in between:

```json
[
  { "kinds": [34236, 22],   "authors": ["3b7fc823...", "f0e2b118..."] },
  { "kinds": [34235, 21],   "authors": ["3b7fc823..."] },
  { "kinds": [1, 6, 16],    "authors": ["a1c9e40d..."] },
  { "kinds": [31922, 31923],"authors": ["f0e2b118..."] }
]
```

The union, sorted newest-first, is the feed. Filter count is the number of
categories a user keeps rather than the number of people they star, so it stays
in the range relays accept however long the lists grow.

Nothing here needs a relay that understands this kind. The work a client does is
merging the responses and paginating across them, holding a cursor per filter and
interleaving — the same thing any feed assembled from several filters already
does. A relay that does understand the kind can return the union directly;
accepting the resolved kinds and authors as request arguments, rather than
reading the user's stored sets, keeps that path working for encrypted entries,
since the client decrypts sets it already holds before asking.

An author's other kinds do not belong in the result. Someone who spotlighted a
musician's tracks and then saw their replies has been given the thing they
specifically declined.

### Kinds that are republished

Some kinds are re-emitted rather than posted once. A [NIP-53](../nips/53.md)
`kind:30311` live event is updated throughout a broadcast — line 52 of that
document expects exactly this — so each update is a new event id on the same
`<kind>:<pubkey>:<d>` coordinate, and a feed sorted by `created_at` fills with
one stream repeating.

Collapse to one entry per coordinate. For ordering, `created_at` marks the last
update rather than the thing that happened, so where the kind carries its own
timestamp that one is better: `30311` has `starts`, and "they went live" is the
entry worth showing rather than "their viewer count changed". Ordering by first
observation is not available — it is the reader's own history rather than a
property of the event, two readers disagree, and a replaceable kind usually has
no earlier version left on the relay to observe.

### Where kinds stop

A category is only as precise as its kinds are. Where one kind carries two
kinds of content, this event cannot separate them and the client continues alone.

The clear case is `kind:1`. A reply is a `kind:1` that carries an `e` tag, so a
filter asking for `1` returns plain notes, quote posts and replies together.
Sets named `notes` and `replies` can list different people, but both resolve to
`kind:1` for anyone in both, and no arrangement of `k` tags divides them.

That is a limit worth stating rather than working around. Fetch what the kinds
select, then divide by tag when rendering — for `kind:1`, on whether a root tag
is present. The alternative, a category vocabulary carrying conditions, would
put a query language in a list.

## Implementation notes

* Someone muted stays out of the feed even when a set still names them.
* Any kind may be listed. This document does not enumerate content kinds and does
  not need revising when new ones appear; a client that learns to render a kind
  can add it to a set the same day.
* A `k` tag naming a kind the client cannot render is skipped, and the set still
  reads.
* The same person may appear in several sets. That is ordinary — one author's
  reels and blogs are two independent preferences — and their events are
  deduplicated by id where categories overlap.
* Sets are `d`-identified, so two clients writing different category names for
  the same idea produce two sets rather than a conflict. Reading both is correct.

### A complete example

One user's whole spotlight: five categories, eight people, five events. One event
carries one `d`, so there is no single document holding all of it — this is the
entire state.

```json
{
  "kind": 30039,
  "content": "",
  "tags": [
    ["d","notes"],
    ["k","1"],
    ["k","6"],
    ["k","16"],
    ["star","3b7fc8231bd4f0e2b1187c3a5e94d0c6a2f8b13e5d7c09a4e6b2f8d1c3a5e7b9"],
    ["star","a1c9e40d7f2b8c5e3a9d1f4b6c8e0a2d5f7b9c1e3a5d7f9b1c3e5a7d9f1b3c5e"],
    ["star","f0e2b1187c3a5e94d0c6a2f8b13e5d7c09a4e6b2f8d1c3a5e7b93b7fc8231bd4"]
  ]
}
{
  "kind": 30039,
  "content": "",
  "tags": [
    ["d","reels"],
    ["k","34236"],
    ["k","22"],
    ["star","3b7fc8231bd4f0e2b1187c3a5e94d0c6a2f8b13e5d7c09a4e6b2f8d1c3a5e7b9"],
    ["star","77c3a5e94d0c6a2f8b13e5d7c09a4e6b2f8d1c3a5e7b93b7fc8231bd4f0e2b11"],
    ["star","5d7c09a4e6b2f8d1c3a5e7b93b7fc8231bd4f0e2b1187c3a5e94d0c6a2f8b13e"]
  ]
}
{
  "kind": 30039,
  "content": "",
  "tags": [
    ["d","music"],
    ["k","32123"],
    ["star","2f8b13e5d7c09a4e6b2f8d1c3a5e7b93b7fc8231bd4f0e2b1187c3a5e94d0c6a"]
  ]
}
```

## Privacy

`p` marks someone an event is addressed to — that is what relays index it for,
and what clients route to notifications. The people in a set are its subject
rather than its audience; a spotlight says nothing to them and is not theirs to
answer.

`star` rather than `p` also keeps `{"#p": ["<pubkey>"]}` from answering who has
spotlighted a person, and for what — [NIP-01](../nips/01.md) spells tag filters as
`#<single-letter>`, so `#star` is not a filter key. `kind:30007` takes the other
option and uses `p`.

That is worth stating accurately rather than overselling. A public entry is
public: anyone with a relay or a firehose can build the reverse index, and the
tag name does not stop them. It decides whether the question is answerable by
accident or only on purpose. The encrypted half is the part that actually
withholds anything.

* A public entry names someone who never agreed to being categorised. A private
  one does not, which is why the private half reads as the better default.
* The reverse view — who spotlighted this author, and how many — is assemblable
  by anyone who wants it, and is nobody's obligation to publish.
* Sets kept on the device publish nothing, at the cost of losing them on a change
  of device. Some users will prefer that trade.
* Relays may decline to serve this kind to anyone but its author.
