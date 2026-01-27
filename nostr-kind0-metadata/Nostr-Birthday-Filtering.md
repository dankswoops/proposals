# Nostr Birthday Filtering

**Related:** [Kind-0 Metadata Proposal](./README.md)

---

## Overview

The `birthday` field in kind-0 metadata is a subjective object. Each of its three properties — `year`, `month`, and `day` — is independently optional, meaning users can share as much or as little as they choose. A user might publish only their birth year, only their month and day, or the full date.

This flexibility creates a challenge: there is no guarantee that a birthday object received from the network contains valid or realistic data. Clients need a consistent way to validate birthday values both before publishing and when rendering profiles from other clients.

This document describes a filtering approach used in ZapHub to solve both problems with a single set of functions.

---

## The Birthday Object

As defined in NIP-24, the birthday field is a JSON object with three optional integer properties:

```json
"birthday": { "year": 1990, "month": 4, "day": 15 }
```

Any property can be omitted:

```json
"birthday": { "month": 4, "day": 15 }
"birthday": { "year": 1990 }
"birthday": {}
```

---

## Validation Rules

### Year

- Must be a valid integer.
- Must not be in the future (no year greater than the current year).
- Must not exceed 126 years in the past, based on the oldest verified human lifespan. This prevents unrealistic values like `"year": 1` or `"year": 1800` from reaching the network.
- If invalid, the year is cleared (treated as omitted).

### Month

- Must be an integer between `1` and `12`.
- Output is zero-padded to two digits (`"4"` becomes `"04"`).
- If invalid, the month is cleared.

### Day

- Must be an integer between `1` and the maximum number of days for the given month.
- The maximum is determined dynamically:
  - **30-day months:** April, June, September, November.
  - **February:** 28 days by default, or 29 if a leap year is confirmed.
  - **All other months:** 31 days.
- **Leap year calculation:** A year is a leap year if it is divisible by 4 **and** not divisible by 100, **unless** it is also divisible by 400. For example, 2000 was a leap year but 1900 was not.
- If no year is provided, February is capped at 28 days. This is a conservative default — the filter cannot confirm a leap year without a year value.
- If invalid, the day is cleared.

### Cross-field dependency

When a user changes their year or month, the day is re-validated against the updated context. For example:

1. A user enters `day: 29`, `month: 2`, `year: 2000` — valid (2000 is a leap year).
2. The user then changes the year to `2001` — the day `29` is no longer valid for February in a non-leap year, so the day is cleared.

This prevents impossible dates like February 29 in a non-leap year from being published.

---

## Reference Implementation

Three standalone functions handle each field. They accept string input (from form fields) and return either a validated string or an empty string (meaning "clear this value").

### validateBirthdayYear

```typescript
export const validateBirthdayYear = (year: string): string => {
  if (!year) return '';
  const yearNum = parseInt(year, 10);
  const currentYear = new Date().getFullYear();
  const minYear = currentYear - 126;
  if (isNaN(yearNum) || yearNum < minYear || yearNum > currentYear) {
    return '';
  }
  return year;
};
```

### validateBirthdayMonth

```typescript
export const validateBirthdayMonth = (month: string): string => {
  if (!month) return '';
  const monthNum = parseInt(month, 10);
  if (isNaN(monthNum) || monthNum < 1 || monthNum > 12) {
    return '';
  }
  return monthNum.toString().padStart(2, '0');
};
```

### validateBirthdayDay

```typescript
export const validateBirthdayDay = (
  day: string,
  month: string,
  year: string
): string => {
  if (!day) return '';
  const dayNum = parseInt(day, 10);
  const monthNum = month ? parseInt(month, 10) : null;
  const yearNum = year ? parseInt(year, 10) : null;

  let maxDays = 31;
  if (monthNum) {
    if ([4, 6, 9, 11].includes(monthNum)) {
      maxDays = 30;
    } else if (monthNum === 2) {
      if (yearNum) {
        const isLeapYear =
          (yearNum % 4 === 0 && yearNum % 100 !== 0) ||
          (yearNum % 400 === 0);
        maxDays = isLeapYear ? 29 : 28;
      } else {
        maxDays = 28;
      }
    }
  }

  if (isNaN(dayNum) || dayNum < 1 || dayNum > maxDays) {
    return '';
  }
  return dayNum.toString().padStart(2, '0');
};
```

---

## Input Fields

The UI presents three side-by-side text inputs within a single container: **YYYY**, **MM**, and **DD**. Each input:

- Accepts only numeric characters (non-digits are stripped on keystroke).
- Enforces a `maxLength` of 4 for year, 2 for month, and 2 for day.
- Runs its validation function `onBlur` (when the user leaves the field), not on every keystroke. This allows the user to type freely without the value being cleared mid-entry.
- When the year or month field loses focus, the day is also re-validated to catch cross-field conflicts.

---

## Dual Use: Publishing and Rendering

These filters serve two purposes:

### Before publishing (outbound)

When a user fills out their profile and submits a kind-0 event, the filter runs on each birthday field before the event is composed. This prevents invalid or impossible dates from being written to the network.

### When rendering (inbound)

When a client receives a kind-0 event from another user, the birthday object may contain values that were set by a client without validation — or values that were valid at the time but are no longer realistic (e.g., a year that now exceeds the 126-year window). The same filter functions can be applied to inbound data to silently ignore or correct invalid fields before rendering them in the UI. This means the client never displays a nonsensical birthday, regardless of what the network delivers.
