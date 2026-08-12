# Homey

`homey` is a Claude Code command for creating, updating, and managing Homey Pro
flows through Homey's local REST API. It is the repository's only
command-only plugin: it has no Codex, Cursor, opencode, or `SKILL.md` surface.

The command contract is
[`commands/homey-flows.md`](commands/homey-flows.md).

## Invoke it

Install the `homey` plugin in Claude Code, then invoke:

```text
/homey:homey-flows
```

Describe the requested automation after the command, for example:

```text
/homey:homey-flows Create a flow that turns on the hallway lights when the
door sensor opens, only between sunset and 23:00.
```

Homey is not available through Codex, Cursor, or opencode in this repository.

## Safety first

This command talks to a real local Homey Pro instance and has write-capable
tools. Treat `POST`, `PUT`, and `DELETE` as live state changes.

Before writing a flow:

1. confirm the target Homey and the intended devices;
2. discover device IDs and available cards;
3. read existing flows when updating or deleting;
4. show or inspect the exact payload and changed fields;
5. confirm destructive deletes or broad updates.

Keep `HOMEY_TOKEN` out of prompts, committed files, screenshots, and logs. Use
a Personal Access Token from `my.homey.app` through the environment variable;
do not paste the token into a command body.

## Connection

The API base is:

```text
http://<HOMEY_IP>/api
```

Authentication uses:

```http
Authorization: Bearer <HOMEY_TOKEN>
```

The command reads the token from `HOMEY_TOKEN`. Set the actual Homey address in
the shell context used for the request; the command file contains the current
local default. Collection and card endpoints require a trailing slash. Flow
ID `PUT` and `DELETE` endpoints intentionally omit it; keep those URL forms
distinct. Collection example:

```sh
TOKEN="$HOMEY_TOKEN"
IP="<HOMEY_IP>"
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "http://$IP/api/manager/devices/device/"
```

Use HTTP only on the trusted local network. Do not expose the token or Homey
API outside that network.

## Required discovery

Discovery always comes before flow construction. List devices and the three
card families:

```sh
TOKEN="$HOMEY_TOKEN"
IP="<HOMEY_IP>"

curl -s -H "Authorization: Bearer $TOKEN" \
  "http://$IP/api/manager/devices/device/"

curl -s -H "Authorization: Bearer $TOKEN" \
  "http://$IP/api/manager/flow/flowcardtrigger/"

curl -s -H "Authorization: Bearer $TOKEN" \
  "http://$IP/api/manager/flow/flowcardcondition/"

curl -s -H "Authorization: Bearer $TOKEN" \
  "http://$IP/api/manager/flow/flowcardaction/"
```

Filter the card results by device ID and capability. Do not invent a card ID,
argument name, device ID, or capability name from a similar device.

## Ordinary flow CRUD

| Method | Endpoint | Body/behavior |
| --- | --- | --- |
| `GET` | `/api/manager/flow/flow/` | List all flows |
| `POST` | `/api/manager/flow/flow/` | Create a flow object at the top level |
| `PUT` | `/api/manager/flow/flow/{id}` | Partial update; send only changed fields |
| `DELETE` | `/api/manager/flow/flow/{id}` | Delete a flow after confirmation |

The create body is not nested under `{ "flow": { ... } }`:

```json
{
  "name": "Flow name",
  "enabled": true,
  "trigger": { },
  "conditions": [],
  "actions": []
}
```

Nesting the object produces `Missing Parameter: flow.name`. Keep the body
top-level and include only fields supported by the discovered API.

## Card formats

### Trigger

Device trigger:

```json
{
  "id": "homey:device:{deviceId}:{triggerId}",
  "uri": "homey:flowcardtrigger:homey:device:{deviceId}:{triggerId}",
  "args": { "threshold": 19 }
}
```

System time/cron triggers replace the device owner with
`homey:manager:cron`.

### Condition

Every condition must include a `group` field. Omitting it can crash the Homey
web UI with `Cannot read properties of undefined (reading 'push')`.

Valid condition groups are `group1`, `group2`, and `group3`:

```json
{
  "id": "homey:manager:cron:time_between",
  "args": { "time1": "08:00", "time2": "17:00" },
  "group": "group1"
}
```

The condition `uri` is optional. `inverted: false` is normal; `inverted: true`
negates the condition.

### Logic condition with a device value

Use a pipe between the device URI and capability name in `droptoken`:

```json
{
  "id": "homey:manager:logic:lt",
  "group": "group1",
  "droptoken": "homey:device:{deviceId}|{capabilityName}",
  "args": { "comparator": 150 }
}
```

Do not use a colon. A colon form displays `Unavailable` in the UI; the pipe
form resolves the capability.

### Action

```json
{
  "id": "homey:device:{deviceId}:{actionId}",
  "uri": "homey:flowcardaction:homey:device:{deviceId}:{actionId}",
  "args": {},
  "group": "then"
}
```

Valid action groups are `then` and `else`.

### Day multiselect

Pass day values as an array of strings:

| Value | Day |
| --- | --- |
| `"0"` | Sunday |
| `"1"` | Monday |
| `"2"` | Tuesday |
| `"3"` | Wednesday |
| `"4"` | Thursday |
| `"5"` | Friday |
| `"6"` | Saturday |

## Light temperature

Homey's `light_temperature` capability uses a `0–1` range mapped linearly in
mireds, not Kelvin:

- `0` is coolest;
- `1` is warmest;
- mireds are `1,000,000 / kelvin`.

For a device with known minimum and maximum mired values:

```text
mired = 1,000,000 / kelvin
homey_value = (mired - min_mired) / (max_mired - min_mired)
```

Common ranges:

| Device family | Range |
| --- | --- |
| Philips Hue White Ambiance | 153–454 mireds, approximately 6536K–2203K |
| IKEA TRADFRI tunable | 250–454 mireds, approximately 4000K–2203K |

For the Hue Ambiance range, approximate values are:

| Target | Mireds | Homey value |
| --- | ---: | ---: |
| 6500K | 154 | 0.00 |
| 4500K | 222 | 0.23 |
| 2700K | 370 | 0.72 |
| 2200K | 454 | 1.00 |

Use the actual device range when available. Treating the range as linear in
Kelvin produces the wrong color temperature.

## Advanced Flows

Advanced flows use:

```text
POST /api/manager/flow/advancedflow/
```

The body contains named cards connected by UUID references:

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
      "x": 200,
      "y": 200,
      "outputSuccess": "{next-uuid}",
      "outputTrue": "{true-uuid}",
      "outputFalse": "{false-uuid}"
    }
  }
}
```

Use only fields and card IDs confirmed by discovery. Validate every referenced
UUID and branch output before posting.

## Common failures

| Mistake | Symptom | Fix |
| --- | --- | --- |
| Missing trailing slash on collection/card URL | `404` | Add `/`; flow ID `PUT`/`DELETE` URLs intentionally omit it |
| Missing condition `group` | Homey UI crashes with `push` of undefined | Add `group1`, `group2`, or `group3` |
| Colon in `droptoken` | UI shows `Unavailable` | Use `device-id\|capability` |
| Body nested under `flow` | `Missing Parameter: flow.name` | Put fields at the top level |
| Invented sensor condition card | Card is unavailable | Use a sensor trigger and a Logic condition with `droptoken` |
| Kelvin-linear temperature mapping | Wrong light output | Convert Kelvin to mireds first |
| Stale device/card discovery | Request fails or targets wrong device | Discover again immediately before writing |
| Wrong Homey or token | Connection/authentication failure | Check local IP, `HOMEY_TOKEN`, and Homey availability without printing the token |

## Source map

- [`commands/homey-flows.md`](commands/homey-flows.md) — command metadata,
  connection defaults, API contracts, payload examples, and pitfalls.
- [`../../README.md`](../../README.md) — installation and harness coverage.
- `.claude-plugin/plugin.json` — Claude Code plugin metadata.

There is no Homey-specific automated validator. Validate changes with a
read-only markdown review, JSON parsing for manifest edits, and an authorized
manual smoke test against a non-production or explicitly selected Homey.
