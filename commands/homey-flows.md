---
description: "Create, update, and manage Homey Pro flows via the local REST API. Use when the user wants to automate devices, create flows, or manage their Homey Pro smart home."
allowed-tools: ["Bash", "Read", "Write", "Edit", "Glob", "Grep", "WebFetch", "WebSearch", "Agent"]
---

# Homey Pro Flow Management

Create and manage flows on Homey Pro via the local REST API.

## Connection

- **Base URL:** `http://{HOMEY_IP}/api` (default: `192.168.1.4`)
- **Auth:** `Authorization: Bearer {TOKEN}` — Personal Access Token from my.homey.app
- **All endpoints require trailing slash** (e.g., `/api/manager/flow/flow/`)
- **Token env var:** `HOMEY_TOKEN`

## Discovery — Always Do First

Before creating flows, discover device IDs and available flow cards:

```bash
TOKEN="$HOMEY_TOKEN"; IP="192.168.1.4"

# List all devices with capabilities
curl -s -H "Authorization: Bearer $TOKEN" "http://$IP/api/manager/devices/device/"

# List available trigger/condition/action cards
curl -s -H "Authorization: Bearer $TOKEN" "http://$IP/api/manager/flow/flowcardtrigger/"
curl -s -H "Authorization: Bearer $TOKEN" "http://$IP/api/manager/flow/flowcardcondition/"
curl -s -H "Authorization: Bearer $TOKEN" "http://$IP/api/manager/flow/flowcardaction/"
```

Filter results by device ID to find relevant cards for each device.

## Flow CRUD

| Method | Endpoint | Body |
|--------|----------|------|
| `POST` | `/api/manager/flow/flow/` | Flow object (top-level, NOT nested under `{ flow: {} }`) |
| `PUT` | `/api/manager/flow/flow/{id}` | Partial update — only changed fields |
| `DELETE` | `/api/manager/flow/flow/{id}` | — |
| `GET` | `/api/manager/flow/flow/` | List all flows |

### Flow Object

```json
{
  "name": "Flow name",
  "enabled": true,
  "trigger": { ... },
  "conditions": [],
  "actions": []
}
```

## Card Formats

### Trigger

```json
{
  "id": "homey:device:{deviceId}:{triggerId}",
  "uri": "homey:flowcardtrigger:homey:device:{deviceId}:{triggerId}",
  "args": { "threshold": 19 }
}
```

System triggers (time/cron): replace `homey:device:{deviceId}` with `homey:manager:cron`.

### Condition

**CRITICAL: Must include `group` field.** Without it, the Homey web UI crashes:
```
TypeError: Cannot read properties of undefined (reading 'push')
```

Valid groups: `"group1"`, `"group2"`, `"group3"`.

```json
{
  "id": "homey:manager:cron:time_between",
  "args": { "time1": "08:00", "time2": "17:00" },
  "group": "group1"
}
```

The `uri` field on conditions is optional.
The `inverted` field works: `false` = normal, `true` = negated ("is NOT").

### Condition with Droptoken (device value reference)

To reference a device capability in a Logic condition, the separator is **pipe `|`**:

```json
{
  "id": "homey:manager:logic:lt",
  "group": "group1",
  "droptoken": "homey:device:{deviceId}|{capabilityName}",
  "args": { "comparator": 150 }
}
```

**CRITICAL:** Use **pipe `|`** NOT colon `:` between device URI and capability name.
- WRONG: `"homey:device:abc123:measure_voc"` → shows "Unavailable" in UI
- RIGHT: `"homey:device:abc123|measure_voc"` → works

### Action

```json
{
  "id": "homey:device:{deviceId}:{actionId}",
  "uri": "homey:flowcardaction:homey:device:{deviceId}:{actionId}",
  "args": {},
  "group": "then"
}
```

Valid groups: `"then"`, `"else"`.

### Day Multiselect

`"0"` = Sunday, `"1"` = Monday, ..., `"6"` = Saturday. Pass as array of strings.

## Light Temperature

Homey `light_temperature` capability (0–1 range):
- **0 = coolest**, **1 = warmest**
- **Mapping is linear in mireds**, NOT Kelvin

### Conversion

```
mired = 1,000,000 / kelvin
homey_value = (mired - min_mired) / (max_mired - min_mired)
```

Common ranges:
- **Philips Hue White Ambiance:** 153–454 mireds (6536K–2203K)
- **IKEA TRADFRI tunable:** 250–454 mireds (4000K–2203K)

| Target K | Mireds | Hue Ambiance % |
|----------|--------|----------------|
| 6500K | 154 | 0.00 |
| 4500K | 222 | 0.23 |
| 2700K | 370 | 0.72 |
| 2200K | 454 | 1.00 |

## Advanced Flows

```
POST /api/manager/flow/advancedflow/
```

```json
{
  "name": "Name",
  "enabled": true,
  "cards": {
    "{uuid}": {
      "type": "trigger|condition|action",
      "ownerUri": "homey:device:{deviceId}",
      "id": "homey:device:{deviceId}:{cardId}",
      "args": {},
      "x": 200, "y": 200,
      "outputSuccess": "{next-uuid}",
      "outputTrue": "{true-uuid}",
      "outputFalse": "{false-uuid}"
    }
  }
}
```

## Common Pitfalls

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing trailing slash on URL | 404 | Add `/` |
| Missing `group` on conditions | UI crash: `push` of undefined | Add `"group": "group1"` |
| Colon in droptoken | "Unavailable" in UI | Use pipe `\|` |
| Body nested under `{ flow: {} }` | "Missing Parameter: flow.name" | Put fields at top level |
| Sensor condition cards | Card unavailable | Sensors only have triggers, use `logic:lt` with droptoken |
| `light_temperature` assumed linear in Kelvin | Wrong color output | Calculate via mireds |
