# Operations Log - Developer Guide

> Append-only Event-Sourced Log for Time-Bound Updates

This document explains how to correctly post updates to `operations.json` using the Epistemic Operators (EO) system.

---

## Quick Reference

| Operator | Code | When to Use |
|----------|------|-------------|
| **INS** | `INS` | Adding NEW content (warming centers, alerts, road closures) |
| **ALT** | `ALT` | MODIFYING existing content (update address, change status) |
| **NUL** | `NUL` | REMOVING content or SCHEDULING removal |
| **SUP** | `SUP` | Adding context/notes WITHOUT changing original data |

---

## Golden Rules

1. **NEVER delete entries** - This is append-only. To remove something, add a `NUL` operation.
2. **NEVER modify existing entries** - Add a new operation instead.
3. **ALWAYS include full sourcing** - Every operation needs `source_url` or `source` in context.
4. **ALWAYS include actor** - Who made this change? Put it in the `frame`.
5. **Generate sequential IDs** - Format: `op_XXX` where XXX is the next number.

---

## Operation Structure

Every operation follows this structure:

```json
{
  "id": "op_XXX",
  "ts": "ISO-8601 timestamp with timezone",
  "op": "INS|ALT|NUL|SUP",
  "target": {
    "id": "unique_identifier_of_what_youre_changing",
    "type": "type_category"
  },
  "context": {
    "table": "logical_table_name",
    "data": { },
    "source_url": "where_this_info_came_from"
  },
  "frame": {
    "actor": "who_made_this_change",
    "reason": "why_this_change_was_made",
    "verified_date": "YYYY-MM-DD",
    "expiry_days": 3
  }
}
```

---

## How to Add Content (INS)

Use `INS` when adding **new** warming centers, road closures, alerts, or any new content.

### Example: Adding a New Warming Center

```json
{
  "id": "op_004",
  "ts": "2026-01-31T14:30:00-06:00",
  "op": "INS",
  "target": {
    "id": "warming_center_hermitage",
    "type": "warming_center"
  },
  "context": {
    "table": "warming_centers",
    "data": {
      "name": "Hermitage Community Center",
      "address": "1234 Example Rd, Hermitage, TN 37076",
      "lat": 36.1900,
      "lng": -86.5800,
      "hasFood": true
    },
    "source_url": "https://www.nashville.gov/news/new-warming-center"
  },
  "frame": {
    "actor": "jane_doe",
    "reason": "New warming center opened per Metro announcement",
    "verified_date": "2026-01-31",
    "expiry_days": 3
  }
}
```

### Example: Adding a New Alert Banner

```json
{
  "id": "op_005",
  "ts": "2026-02-01T09:00:00-06:00",
  "op": "INS",
  "target": {
    "id": "flood_alert_banner",
    "type": "banner"
  },
  "context": {
    "table": "alerts",
    "data": {
      "component": "floodAlertBanner",
      "message": "Flash flood warning in effect",
      "severity": "severe"
    },
    "source_url": "NWS Nashville"
  },
  "frame": {
    "actor": "john_smith",
    "reason": "Flash flood warning issued by NWS",
    "expires_at": "2026-02-02T18:00:00-06:00"
  }
}
```

---

## How to Modify Content (ALT)

Use `ALT` when **changing** existing content - updating an address, changing hours, correcting information.

### Example: Updating a Warming Center's Hours

```json
{
  "id": "op_006",
  "ts": "2026-01-31T16:00:00-06:00",
  "op": "ALT",
  "target": {
    "id": "warming_center_hermitage",
    "field": "hours"
  },
  "context": {
    "table": "warming_centers",
    "old": "24 hours",
    "new": "6am - 10pm",
    "source_url": "https://www.nashville.gov/news/updated-hours"
  },
  "frame": {
    "actor": "jane_doe",
    "reason": "Hours changed per Metro update"
  }
}
```

### Example: Correcting a Road Closure Location

```json
{
  "id": "op_007",
  "ts": "2026-01-31T11:00:00-06:00",
  "op": "ALT",
  "target": {
    "id": "road_closure_charlotte",
    "field": "location"
  },
  "context": {
    "table": "road_closures",
    "old": "Charlotte Ave at I-40",
    "new": "Charlotte Ave at I-440",
    "source_url": "Hub Nashville correction"
  },
  "frame": {
    "actor": "traffic_ops",
    "reason": "Location was incorrectly reported"
  }
}
```

---

## How to Remove Content (NUL)

Use `NUL` to **remove** content OR **schedule** its removal. This is the only way to delete.

### Example: Road Closure Cleared (Immediate Removal)

```json
{
  "id": "op_008",
  "ts": "2026-01-31T15:00:00-06:00",
  "op": "NUL",
  "target": {
    "id": "road_closure_charlotte"
  },
  "context": {
    "table": "road_closures",
    "reason": "Road reopened - ice cleared",
    "source_url": "Hub Nashville / TDOT"
  },
  "frame": {
    "actor": "traffic_ops",
    "effective_immediately": true
  }
}
```

### Example: Scheduling Banner Removal (Future Expiration)

```json
{
  "id": "op_009",
  "ts": "2026-01-31T10:00:00-06:00",
  "op": "NUL",
  "target": {
    "id": "shelter_alert_banner"
  },
  "context": {
    "table": "alerts",
    "reason": "Storm emergency ending, schedule banner removal"
  },
  "frame": {
    "actor": "emergency_mgmt",
    "effective_at": "2026-02-03T00:00:00-06:00",
    "reversible": true
  }
}
```

### Example: Closing a Warming Center

```json
{
  "id": "op_010",
  "ts": "2026-02-01T08:00:00-06:00",
  "op": "NUL",
  "target": {
    "id": "warming_center_hermitage"
  },
  "context": {
    "table": "warming_centers",
    "reason": "Center closed - storm passed",
    "source_url": "Metro Parks announcement"
  },
  "frame": {
    "actor": "parks_dept",
    "effective_immediately": true
  }
}
```

---

## How to Add Context (SUP)

Use `SUP` to **layer additional information** onto existing content without changing it.

### Example: Adding a Note to a Warming Center

```json
{
  "id": "op_011",
  "ts": "2026-01-31T12:00:00-06:00",
  "op": "SUP",
  "target": {
    "id": "warming_centers_dataset"
  },
  "context": {
    "table": "warming_centers",
    "overlay": {
      "note": "All centers now providing hot meals until 8pm",
      "applies_to": "all"
    },
    "source_url": "Metro Social Services"
  },
  "frame": {
    "actor": "social_services",
    "expires_at": "2026-02-02T00:00:00-06:00"
  }
}
```

---

## Required Fields Checklist

Before submitting an operation, verify:

- [ ] `id` - Unique, sequential (check the last `op_XXX` number)
- [ ] `ts` - Current timestamp in ISO-8601 with timezone (`-06:00` for Central)
- [ ] `op` - One of: `INS`, `ALT`, `NUL`, `SUP`
- [ ] `target.id` - Identifies what you're operating on
- [ ] `context.table` - Logical grouping (`warming_centers`, `road_closures`, `alerts`)
- [ ] `context.source_url` - Where did this information come from?
- [ ] `frame.actor` - Your identifier (name, role, or system)
- [ ] `frame.reason` - Why is this change being made?

---

## Common Mistakes to Avoid

### Wrong: Deleting an entry from the JSON

```diff
- DO NOT DO THIS
- Just delete the entry from operations array
```

### Right: Add a NUL operation

```json
{
  "op": "NUL",
  "target": { "id": "the_thing_to_remove" },
  "context": { "reason": "why its being removed" }
}
```

---

### Wrong: Modifying an existing operation

```diff
- DO NOT DO THIS
- Go back and edit op_003 to change the data
```

### Right: Add an ALT operation

```json
{
  "op": "ALT",
  "target": { "id": "op_003_target", "field": "what_changed" },
  "context": { "old": "previous_value", "new": "new_value" }
}
```

---

### Wrong: Missing source

```json
{
  "op": "INS",
  "context": {
    "data": { "name": "New Center" }
  }
}
```

### Right: Include source

```json
{
  "op": "INS",
  "context": {
    "data": { "name": "New Center" },
    "source_url": "https://nashville.gov/announcement"
  }
}
```

---

## Timestamp Format

Always use ISO-8601 with Central timezone:

```
2026-01-31T14:30:00-06:00
```

- `-06:00` = Central Standard Time (CST)
- `-05:00` = Central Daylight Time (CDT, during summer)

To get current timestamp in terminal:
```bash
date -Iseconds
```

---

## Target Types Reference

| Type | Used For |
|------|----------|
| `warming_center` | Individual warming center location |
| `location_dataset` | Collection of locations (warming centers, road closures) |
| `road_closure` | Individual road closure |
| `banner` | UI alert/notification banner |
| `alert` | Emergency alert content |

---

## Context Tables Reference

| Table | Content Type |
|-------|--------------|
| `warming_centers` | Warming center locations |
| `road_closures` | Road closure information |
| `alerts` | Alert banners and notifications |
| `shelter` | Shelter information |

---

## Frame Fields Reference

| Field | Type | Description |
|-------|------|-------------|
| `actor` | string | Who made this change |
| `reason` | string | Why this change was made |
| `verified_date` | string | Date information was verified (YYYY-MM-DD) |
| `expiry_days` | number | Days until data becomes stale |
| `expires_at` | string | Specific expiration timestamp |
| `effective_at` | string | When change takes effect (for scheduled changes) |
| `effective_immediately` | boolean | Change takes effect now |
| `reversible` | boolean | Whether this NUL can be undone |
| `source` | string | Alternative to source_url for non-URL sources |

---

## Workflow Example: Full Lifecycle

### Day 1: New warming center opens

```json
{ "op": "INS", "target": { "id": "wc_new_location" }, ... }
```

### Day 2: Hours change

```json
{ "op": "ALT", "target": { "id": "wc_new_location", "field": "hours" }, ... }
```

### Day 3: Add temporary note

```json
{ "op": "SUP", "target": { "id": "wc_new_location" }, "context": { "overlay": { "note": "Closed 2-4pm for cleaning" } }, ... }
```

### Day 5: Center closes

```json
{ "op": "NUL", "target": { "id": "wc_new_location" }, "context": { "reason": "Storm passed, center closed" }, ... }
```

---

## Sync with index.html

After updating `operations.json`, the corresponding constants in `index.html` should be updated to reflect the current state. The operations log serves as the authoritative audit trail and source of truth for all changes.

Key constants to sync:
- `SHELTER_ALERT_EXPIRATION` (line ~2176)
- `warmingCenters` array (line ~3814)
- `WARMING_CENTERS_VERIFIED_DATE` (line ~3811)
- `roadClosures` array (line ~3827)
- `ROAD_CLOSURES_VERIFIED_DATE` (line ~3824)

---

*Last updated: 2026-01-30*
