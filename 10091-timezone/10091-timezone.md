NIP-XX
======

User Time Offset
----------------

`draft` `optional` `kind:10091` `replaceable`

## Abstract

A replaceable event carrying one number: how far a user's clock sits from UTC.

It lets clients show what time it is for someone, and lay several people's days
on one timeline to find a meeting slot, without anyone publishing where they
live. It carries no time zone name, no country, and no daylight-saving rule.

## Event

```json
{
  "kind": 10091,
  "content": "",
  "tags": [
    ["utc_offset", "-360"]
  ]
}
```

* `utc_offset` — the user's **standard** offset (the one their region uses when
  daylight saving is not in effect), in **minutes east of UTC**, as a decimal
  integer. Negative is west.

The value is a multiple of 15 in the range `-720`…`+840`. Minutes rather than
`±HH:MM` so there is one representation and no sign-parsing surprises; note that
IANA's `Etc/GMT+6` means UTC−06:00, which is the kind of inversion this avoids.

`content` is unused. Treat a missing, non-integer or out-of-range value the same
as no event at all — unknown, rather than a default of zero.

### Withdrawing

A replaceable event cannot be unsent, so clearing the value means publishing a
newer one that carries nothing:

```json
{
  "kind": 10091,
  "content": "",
  "tags": []
}
```

The newest event wins, so this overwrites the old value wherever it reached.

JSON `null` is not available here. [NIP-01](../nips/01.md) gives the id
serialization over "tags, as an array of arrays of **non-null strings**", so a
null inside a tag is invalid: it fails to deserialize wherever tags are typed as
strings, and risks an id that will not reproduce. An empty tag list says the same
thing and is valid everywhere.

A client that instead sends an unparseable value such as `["utc_offset", "null"]`
means the same thing in a lumpier way. Readers already treat anything they cannot
parse as unknown, so it costs nothing to accept.

A [NIP-09](../nips/09.md) deletion request may be sent alongside, but it reaches
only the relays that receive it, whereas this replacement travels the same path
the original did.

## Values

37 offsets are in use. The value space itself is just the 15-minute grid, so a new legal
offset needs no change here. Population counts are rough magnitudes, there to
show the size of the crowd a value places someone in.

| number | offset | minutes | used by | off-hour | population count | secops suggest |
|--------|--------|---------|---------|----------|------------------|---------|
| 1 | −11:00 | `-660` | American Samoa, Niue |  | ~50 k | −10:00 |
| 2 | −10:00 | `-600` | Hawaii, Tahiti, Cook Is. |  | ~1.7 M |  |
| 3 | −09:30 | `-570` | Marquesas | ✓ | ~9 k | −10:00 |
| 4 | −09:00 | `-540` | Alaska |  | ~735 k | −08:00 |
| 5 | −08:00 | `-480` | US/Canada Pacific, Baja California |  | ~64 M |  |
| 6 | −07:00 | `-420` | US/Canada Mountain, Arizona, NW Mexico |  | ~38 M |  |
| 7 | −06:00 | `-360` | US Central, Mexico, Central America |  | ~245 M |  |
| 8 | −05:00 | `-300` | US Eastern, Colombia, Peru, Ecuador |  | ~315 M |  |
| 9 | −04:00 | `-240` | Caribbean, Venezuela, Chile, Bolivia |  | ~90 M |  |
| 10 | −03:30 | `-210` | Newfoundland | ✓ | ~510 k | −03:00 |
| 11 | −03:00 | `-180` | Brazil, Argentina, Uruguay |  | ~250 M |  |
| 12 | −02:00 | `-120` | Greenland, Fernando de Noronha |  | ~60 k | −03:00 |
| 13 | −01:00 | `-60` | Azores, Cape Verde |  | ~830 k | ±00:00 |
| 14 | ±00:00 | `0` | UK, Ireland, Portugal, West Africa |  | ~275 M |  |
| 15 | +01:00 | `60` | Central Europe, West & Central Africa |  | ~900 M |  |
| 16 | +02:00 | `120` | Eastern Europe, Egypt, Israel, South Africa |  | ~400 M |  |
| 17 | +03:00 | `180` | Moscow, Türkiye, Saudi Arabia, East Africa |  | ~700 M |  |
| 18 | +03:30 | `210` | Iran only | ✓ | ~89 M |  |
| 19 | +04:00 | `240` | UAE, Georgia, Armenia, Azerbaijan |  | ~37 M |  |
| 20 | +04:30 | `270` | Afghanistan only | ✓ | ~42 M |  |
| 21 | +05:00 | `300` | Pakistan, Uzbekistan, Kazakhstan (west) |  | ~315 M |  |
| 22 | +05:30 | `330` | India, Sri Lanka | ✓ | ~1.45 B |  |
| 23 | +05:45 | `345` | Nepal only | ✓ | ~30 M |  |
| 24 | +06:00 | `360` | Bangladesh, Bhutan, Kyrgyzstan |  | ~195 M |  |
| 25 | +06:30 | `390` | Myanmar, Cocos Is. | ✓ | ~54 M |  |
| 26 | +07:00 | `420` | Thailand, Vietnam, western Indonesia |  | ~380 M |  |
| 27 | +08:00 | `480` | China, Philippines, Malaysia, Western Australia |  | ~1.65 B |  |
| 28 | +08:45 | `525` | Eucla | ✓ | ~200 | +09:00 |
| 29 | +09:00 | `540` | Japan, Korea, eastern Indonesia |  | ~225 M |  |
| 30 | +09:30 | `570` | Adelaide, Darwin | ✓ | ~2.1 M |  |
| 31 | +10:00 | `600` | Eastern Australia, PNG, Vladivostok |  | ~34 M |  |
| 32 | +10:30 | `630` | Lord Howe Is. | ✓ | ~400 | +10:00 |
| 33 | +11:00 | `660` | Solomon Is., Vanuatu, Magadan |  | ~2.3 M |  |
| 34 | +12:00 | `720` | New Zealand, Fiji, Kamchatka |  | ~6.7 M |  |
| 35 | +12:45 | `765` | Chatham Is. | ✓ | ~600 | +12:00 |
| 36 | +13:00 | `780` | Samoa, Tonga, Tokelau |  | ~330 k | +12:00 |
| 37 | +14:00 | `840` | Line Islands |  | ~8.5 k | +12:00 |

`-720` is legal but uninhabited. `+13:00` and `+14:00` are not duplicates of
`-11:00` and `-10:00`: the clock matches but the calendar date is a day ahead, so
folding them together puts Samoa and Kiribati on the wrong day.

A `suggest` entry appears wherever the population is **under a million**, and
points at the nearest offset with more than a million people. Offering it is a
courtesy rather than a requirement: it trades 15 minutes to 2 hours of accuracy
for a much larger crowd, so the cost is worth showing alongside it.

The threshold, not the shape of the value, decides this. Several off-hour offsets
carry no suggestion because their populations are large — `+05:30` holds about
1.45 billion people, and rounding it would mean a smaller crowd *and* a worse
clock. Row 37 is the awkward case: the Line Islands are two hours from the
nearest million people, so the trade there is a poor one.

## Deriving the value

The device knows the answer; sample both halves of the year and keep the
westward one, which is the standard offset:

```js
const y = new Date().getFullYear();
// getTimezoneOffset() returns minutes WEST, the opposite sign to this event.
const jan = new Date(y, 0, 1).getTimezoneOffset();
const jul = new Date(y, 6, 1).getTimezoneOffset();
const utcOffset = -Math.max(jan, jul);   // e.g. -360
```

Reading the offset "right now" instead would capture the daylight-saving value
for part of the year, which is both wrong as a standard offset and a hint that
the user observes daylight saving.

## Rendering

Daylight saving is not in the event, so the reader resolves it from its own
tzdb at display time. No zone name is sent or stored; this is a local lookup
against data the reader already ships.

The procedure, runnable with `Intl` alone:

1. Collect every IANA zone whose STANDARD offset equals the published value. A
   zone's standard offset is the westward (smaller) of its January and July
   offsets, the same rule `Deriving the value` uses, so the grouping is derived,
   never a baked-in table.
2. Read each of those zones' offset at the current instant.
3. If they all agree, that value is the current offset: show it. This covers
   both the offsets no region shifts and the offsets every region shifts
   together.
4. If they disagree, the offset spans DST and non-DST regions, so the true value
   is `X` or `X+1` for part of the year. Show one value and MARK IT APPROXIMATE;
   the error is never worse than an hour. Which value to show is a client
   choice, because none is correct for everyone at a mixed offset. A reasonable
   rule, and the one ZapHub uses, is the current time of the most-populous
   DST-observing region where one dominates (so −06:00 renders as US Central),
   and the stored standard value otherwise (so −04:00 renders as Venezuela, its
   larger non-observing side).

The three outcomes, with rough magnitudes **as of IANA tzdb 2025b and
illustrative only**: the assignment is a function of current DST law, which
changes yearly and moves offsets between buckets. Brazil ended DST in 2019,
Mexico in 2022, Iran in 2022, and Fiji suspended it in 2021, each un-mixing an
offset that used to be a band. A client MUST therefore derive the bucket at
render time, never hard-code this list.

* **exact, year-round.** No region at the offset observes DST, so the clock is
  knowable with no seasonal term. The majority of offsets, including the largest
  populations on earth (`+05:30` India, `+08:00` China, `+09:00` Japan/Korea).
* **exact, with a uniform seasonal shift.** The whole offset shifts together,
  so add the shift in season. Newfoundland (`−03:30`, +1h), Lord Howe (`+10:30`,
  +30m), Chatham (`+12:45`, +1h).
* **±1h.** Some regions at the offset observe DST and some do not, so a
  one-hour band is the honest value. `−06:00` (US Central vs Saskatchewan,
  Mexico, Central America), `±00:00` (UK/Ireland/Portugal vs true-GMT West
  Africa), `+01:00` (Central Europe vs Nigeria and West Africa), and the other
  mixed offsets sit here. `−06:00` and `−04:00` also mix northern and southern
  observers, so they never fully agree in either season.

The one-hour ceiling is structural, not incidental: of the ~420 IANA zones, the
large majority do not shift at all, the rest shift exactly 60 minutes, Lord Howe
shifts 30, and only Antarctica/Troll shifts 120 for a single research station,
which no inhabited offset depends on. So a reader that resolves to the wrong side
of a mixed band is wrong by an hour at most, and only for part of the year.

### Comparing days

For planning across several people, lay each person's conventional waking day —
roughly 08:00 to 22:00 local — on a shared UTC timeline and take the overlap.
That band is an assumption about people rather than something the user stated, so
it belongs in the UI as a suggestion, not as their availability. Someone who
wants to publish when they are actually reachable is making a different
statement, which this event does not carry.

## Implementation notes

* Republish only when the underlying fact changes — a move, or a change in the
  law. There is nothing to update at a daylight-saving transition, and a client
  that republished twice a year would leak its user's transition dates, which
  identify their rule set.
* Mapping the value back to a place undoes the point of it. Several offsets
  resolve to a single country, so a flag beside the name republishes exactly what
  choosing this event over a zone name withheld.
* Absence means unknown, and `0` is a legitimate value — row 14, a few hundred
  million people. Using it as the stand-in for "not published" makes a silent
  account indistinguishable from someone who really is at UTC+0, so check whether
  the tag is there before testing what it holds. `Number(tag[1]) || fallback`
  reads a real `0` as missing in JavaScript and Python, and the string `"0"` is
  falsy in Perl and PHP, so `if ($tag[1])` skips it there before any conversion.
* The value changes at most once in a lifetime at one location, so it caches
  indefinitely. When rendering many people, batch by `authors` in one filter.

## Security Considerations

The whole design is a privacy tradeoff, so the threat model is the spec.

**A zone name is a location; an offset is a longitude band.** A zone answers
"where are you" in order to answer "when are you", and only the second was
asked. `America/Costa_Rica` names a country and `Australia/Eucla` names about
two hundred people. An offset is coarser: `−06:00` spans dozens of zones across
three countries. It is not nothing. It gives a band of longitude and, for the
rare values flagged in the `suggest` column, effectively a country, which is why
the UI offers the busier neighbor. For an account that posts, the offset is close
to already public, since the timestamps on its events trace its waking day; it is
a genuinely new disclosure only for accounts that read without posting.

**Standard offset, not current offset, and that is the load-bearing choice.**
Publishing the offset in effect right now would carry the daylight-saving bit,
and republishing it twice a year at each transition would leak the user's
transition DATES. Those dates identify the exact DST rule the user follows, and a
rule set narrows the region far more than the raw offset does: it is the
difference between "somewhere on the −06:00 band" and "a place that springs
forward on the US schedule". So a client MUST publish only the standard offset,
and MUST republish only when the underlying fact changes, a move or a change in
the law, never on a DST transition. A client that re-derived and re-published the
value on a schedule would reintroduce exactly the leak this event is shaped to
avoid.

**DST is resolved by the reader, not carried in the event.** Because the event
omits the rule, each reader applies daylight saving from its own tzdb (see
Rendering). No place data leaves the user to make the clock correct; the cost is
the ±1h band on mixed offsets, which is the right side of the trade.

**Do not map the value back to a place.** Rendering a flag, a country name, or a
single city beside the clock re-discloses precisely what choosing an offset over
a zone withheld. Several offsets resolve to one country, so this is a real
regression, not a cosmetic one. The representative zone a reader uses internally
to resolve DST (Rendering, step 4) is an implementation detail and MUST NOT be
surfaced as the user's location.

**Absence is not zero.** `0` is a real offset used by a few hundred million
people, so a client MUST test whether the tag is present before reading it.
Collapsing "no event" into UTC+0 makes a silent account indistinguishable from
one that really is at UTC+0, and `Number(tag[1]) || fallback` misreads a real `0`
as missing in several languages (see Implementation notes).

**Publishing is opt-in.** The value fits as something a user turns on, not
something filled in at signup or inferred from an IP address. A client that
derived it silently at onboarding would be publishing a location signal the user
never chose to share.
