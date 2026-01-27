# Proposal: Three New Nostr Kind-0 Metadata Fields

**Author:** Adam Johnson, @dankswoops
**Date:** January 2026
**Status:** Draft
**Applies to:** [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) / [NIP-24](https://github.com/nostr-protocol/nips/blob/master/24.md) (Kind-0 Metadata)

---

## Summary

This proposal introduces three new optional fields to Nostr kind-0 (user metadata) events:

| Field      | Type                   | Description                                              |
|------------|------------------------|----------------------------------------------------------|
| `status`   | `integer`              | Relationship status indicator (0-7)                      |
| `partner`  | `string[]`             | An array of hex public keys representing partners        |
| `kin`      | `[string, integer][]`  | An array of tuples: `[hex pubkey, relationship type]`    |

All three fields are optional. Omitting a field means the user has chosen not to disclose that information. Clients that do not recognize these fields can safely ignore them, as is standard with kind-0 metadata.

---

## Background: What is Nostr Kind-0 Metadata?

In the Nostr protocol, a **kind-0 event** is a special replaceable event that stores a user's public profile metadata. The event content is a JSON string containing fields such as `name`, `display_name`, `about`, `picture`, `banner`, `website`, `nip05`, and `lud16`. After the initial spec was created, two additional fields were defined across [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) and [NIP-24](https://github.com/nostr-protocol/nips/blob/master/24.md): `birthday` and `bot`. This proposal appends `status`, `partner`, and `kin` to that list, extending the set of standardized metadata fields available to clients.

Because kind-0 content is an open JSON object, clients are free to include additional fields. Unrecognized fields are ignored by clients that do not support them, making the format naturally extensible. This proposal follows that convention.

---

## Field Definitions

### `status` (integer, 0-7)

A standardized integer representing the user's relationship status.

| Value | Meaning             |
|-------|---------------------|
| `0`   | Unspecified         |
| `1`   | Single              |
| `2`   | In a relationship   |
| `3`   | Engaged             |
| `4`   | Married             |
| `5`   | Divorced            |
| `6`   | Widowed             |
| `7`   | Open relationship   |

Setting `status` to `0` is semantically equivalent to omitting the field — both mean the user has not specified a relationship status. This allows clients to explicitly reset a previously set status back to unspecified without removing the field from the JSON object.

Using an integer rather than a freeform string allows clients to localize the label into any language and maintain consistency across the ecosystem. The range is intentionally compact; future proposals may extend beyond `7` if additional statuses are needed.

### `partner` (string array)

An array of 64-character hex public keys identifying the user's partner(s). The first entry is considered the **primary partner**. Clients may choose to render one, some, or all entries depending on their UI.

This field is meaningful when `status` is `2`, `3`, `4`, or `7`, but clients should not enforce that restriction.

Hex format is used to align with the Nostr protocol's native key representation as defined in [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md). Clients that receive an npub-encoded value should still attempt to decode it, but implementations should always write hex. The bech32 npub encoding adds computational overhead at scale and is intended as a display format, not a storage format.

### `kin` (array of tuples)

An array representing family connections. Each entry is a tuple of `[pubkey, type]` where `pubkey` is a 64-character hex public key and `type` is an integer from the table below. Use type `0` when the relationship is unspecified.

This uniform tuple format ensures consistent parsing across all languages and runtimes. There are no mixed types in the array.

Clients SHOULD NOT write more than **50 kin entries**. A kind-0 event with all standard fields populated and 50 typed kin entries stays under 8 KB, which is within the content limits of most relays as defined by [NIP-11](https://github.com/nostr-protocol/nips/blob/master/11.md). Exceeding this may cause the event to be rejected by restrictive relays.

#### Kin relationship types

| Value | Meaning              |
|-------|----------------------|
| `0`   | Unspecified          |
| `1`   | Parent               |
| `2`   | Child                |
| `3`   | Sibling              |
| `4`   | Grandparent          |
| `5`   | Grandchild           |
| `6`   | Uncle / Aunt         |
| `7`   | Nephew / Niece       |
| `8`   | Cousin               |
| `9`   | In-law               |
| `10`  | Step-relative        |
| `11`  | Godparent / Godchild |
| `12`  | Guardian / Ward      |

#### Examples

Unspecified relationships:
```json
"kin": [
  ["aabb...64-char-hex-pubkey", 0],
  ["ccdd...64-char-hex-pubkey", 0]
]
```

With relationship types:
```json
"kin": [
  ["aabb...64-char-hex-pubkey", 1],
  ["ccdd...64-char-hex-pubkey", 3],
  ["eeff...64-char-hex-pubkey", 0]
]
```

In the above example: the first entry is a parent, the second is a sibling, and the third is unspecified kin.

As with `partner`, hex is the required storage format. Clients should convert npub input to hex before writing.

---

## Example: Kind-0 Event Content

A complete kind-0 metadata JSON object including existing [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md)/[NIP-24](https://github.com/nostr-protocol/nips/blob/master/24.md) fields alongside the proposed fields:

```json
{
  "name": "alice",
  "display_name": "Alice",
  "about": "Building on Nostr.",
  "picture": "https://example.com/alice.jpg",
  "banner": "https://example.com/banner.jpg",
  "website": "https://alice.dev",
  "nip05": "alice@alice.dev",
  "lud16": "alice@getalby.com",
  "birthday": { "year": 1990, "month": 4, "day": 15 },
  "bot": false,
  "status": 4,
  "partner": [
    "3bf0c63fcb93463407af97a5e5ee64fa883d107ef9e558472c4eb9aaaefa459d"
  ],
  "kin": [
    ["a2b4c6d8e0f1234567890abcdef1234567890abcdef1234567890abcdef123456", 1],
    ["b3c5d7e9f1a2345678901bcdef2345678901bcdef2345678901bcdef23456789a", 3],
    ["c4d6e8f0a1b2345678902cdef3456789012cdef3456789012cdef345678901bcd", 0]
  ]
}
```

In this example, Alice is married and has listed one primary partner. Her kin array identifies a parent (`1`), a sibling (`3`), and an unspecified family member (`0`). The `birthday` and `bot` fields use the existing [NIP-24](https://github.com/nostr-protocol/nips/blob/master/24.md) definitions.

Clients that do not support these fields will simply ignore `status`, `partner`, and `kin` while rendering the rest of the profile normally.

---

## Benefits

### Richer identity without complexity
Nostr metadata currently covers names, images, and verification. These three fields extend that to personal relationships, which is a fundamental part of how people represent themselves online, without introducing complex schemas or new event kinds.

### Standardization prevents fragmentation
Without a standard, individual clients will invent their own incompatible fields for the same purpose. A simple shared convention means a user's relationship status set in one client is visible in every client that adopts the standard.

### Integer-based status enables localization
Using integers `0-7` rather than English strings means clients can translate relationship labels into any language. A Japanese client and a Portuguese client both read `"status": 4` and display the appropriate word for "Married." The same applies to kin relationship types — `1` means "Parent" regardless of locale.

### Native hex format avoids unnecessary overhead
Storing `partner` and `kin` as hex public keys aligns with the protocol's native key format used in event tags, filters, and signatures. This avoids the computational overhead of bech32 encoding and decoding at scale, and ensures these fields can be used directly without conversion.

### Graceful degradation
All fields are optional. Clients that do not implement this proposal continue to work without changes. There is no migration burden and no breaking change. This follows the same pattern that [NIP-24](https://github.com/nostr-protocol/nips/blob/master/24.md) used to extend kind-0 with `birthday`, `bot`, and other optional fields.

---

## Design Decisions

- **Why not a new event kind?** Relationship metadata is inherently part of a user's profile. Splitting it into a separate event kind would add complexity for clients and require additional subscriptions.
- **Why hex and not npub?** Hex is the native key format throughout the Nostr protocol — event tags, filters, and signatures all use hex. Storing hex avoids bech32 encode/decode overhead on every read and write. The npub format is a display-layer convenience; clients should convert to npub only when rendering for users. Implementations that receive an npub value should decode it gracefully, but should always write hex.
- **Why is `partner` an array?** A singular string would conflict with `status: 7` (open relationship), where multiple partners are expected. Making `partner` an array with the first entry as the primary partner handles all relationship types consistently. Clients may render one or all entries at their discretion.
- **Why uniform tuples for `kin`?** An earlier draft allowed mixed types (bare strings and tuples) in the `kin` array. This creates parsing complexity in strongly-typed languages like Go, Rust, and Swift, where each element requires runtime type-checking. Using a consistent `[pubkey, type]` tuple with `0` for unspecified eliminates this problem and makes the format trivial to parse in any language.
- **Why integers for `status`?** Integers are compact, language-neutral, and unambiguous. A freeform string would lead to inconsistencies (`"single"` vs `"Single"` vs `"soltero"`).

---

## Privacy

All kind-0 metadata is **public and unencrypted** by design. The fields introduced in this proposal follow the same model — users who include `status`, `partner`, or `kin` in their profile are choosing to disclose that information publicly, just as they choose to disclose their name, photo, or bio.

Users who do not wish to share relationship or family information should simply omit these fields. Omission carries no semantic meaning beyond "not disclosed."

For use cases that require **private** relationship metadata, [NIP-81 (Contact Cards, PR #761)](https://github.com/nostr-protocol/nips/pull/761) proposes a separate event kind with [NIP-44](https://github.com/nostr-protocol/nips/blob/master/44.md) encryption. These two approaches are complementary: this proposal handles public profile metadata; NIP-81 handles private relationship graphs.

---

## Prior Art

### NIP-81: Contact Cards ([PR #761](https://github.com/nostr-protocol/nips/pull/761))

Proposed by [@vitorpamplona](https://github.com/vitorpamplona), NIP-81 defines **kind 30382** events for expressing relationships between two pubkeys using encrypted content and an `n` tag for categorization (e.g. "follow", "friend", "patient"). It is implemented in [Amethyst](https://github.com/vitorpamplona/amethyst) and remains open and unmerged.

**How this proposal differs:**

| | This proposal | NIP-81 |
|---|---|---|
| **Where** | Fields inside kind-0 content JSON | Separate event kind (30382) |
| **Visibility** | Public (same as all kind-0 data) | Encrypted via [NIP-44](https://github.com/nostr-protocol/nips/blob/master/44.md) |
| **Scope** | Profile-level summary (status, partner, family) | Per-relationship granularity |
| **Complexity** | No new event kinds, subscriptions, or encryption | Requires NIP-44, new subscriptions, per-contact events |
| **Use case** | Public "about me" metadata | Private relationship management |

These approaches solve different problems. A user may publicly list their partner in kind-0 (this proposal) while also maintaining a private, granular contact graph via NIP-81.

### Issue [#261](https://github.com/nostr-protocol/nips/issues/261): Kind-0 Metadata Contention

This discussion documents a known limitation of kind-0: because it is a [replaceable event](https://github.com/nostr-protocol/nips/blob/master/01.md), only the latest version survives on each relay. If a client that does not recognize `status`, `partner`, or `kin` publishes a kind-0 update, those fields may be lost. [NIP-24](https://github.com/nostr-protocol/nips/blob/master/24.md) instructs clients to preserve unknown fields when updating kind-0, but not all clients comply. This is an ecosystem-wide issue shared by every kind-0 extension (`birthday`, `bot`, `banner`, etc.) and is not unique to this proposal.

---

## Compatibility

- **Backward compatible:** Clients that do not recognize these fields will ignore them per existing Nostr conventions.
- **Forward compatible:** The integer ranges for `status` (currently 0-7) and kin types (currently 0-12) can be extended in future proposals without breaking existing implementations. Clients SHOULD treat unrecognized integers as "unspecified."
- **Relay compatible:** These fields add minimal bytes to the kind-0 content, well within [NIP-11](https://github.com/nostr-protocol/nips/blob/master/11.md) relay size limits.

---

## Related

- [Nostr Birthday Filtering](./Nostr-Birthday-Filtering.md) — A reference for working with the `birthday` object, including input validation, leap year handling, and filtering invalid dates from the network.
