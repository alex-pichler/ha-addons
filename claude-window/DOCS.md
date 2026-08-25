# Claude Window Keeper

Fires a throwaway Claude Code prompt on a schedule so a subscription usage
window is already open — and partly spent — by the time you start working.

## One-time login

Claude Code's browser OAuth flow cannot complete inside this container, so
authentication is done with a long-lived token generated elsewhere.

1. On any machine with a browser and Claude Code installed, run:

   ```
   claude setup-token
   ```

2. Complete the browser login. The token is printed **once** — copy it.
3. Paste it into the add-on's `oauth_token` option and start the add-on.

The token is valid for about a year. When it expires the add-on will start
logging `error` in `sensor.claude_keepalive_status`; repeat the steps above.

Requires a Claude Pro or Max subscription. An API key would be billed
separately and would not open a subscription window, so the add-on
deliberately strips `ANTHROPIC_API_KEY` and `ANTHROPIC_AUTH_TOKEN` from its
environment before invoking Claude.

## Options

| Option | Default | Meaning |
| --- | --- | --- |
| `oauth_token` | — | Output of `claude setup-token`. |
| `window_hours` | `5` | Length of a usage window. Only used for the sensors and for keepalive spacing. |
| `margin_minutes` | `5` | Safety pad subtracted when logging window end, and added to the keepalive interval. |
| `daily_primes` | `["08:00","13:00","18:00","23:00"]` | Fixed local times to ping. Quote them — bare `08:00` is not a YAML string. |
| `weekdays_only` | `false` | Skip Saturday and Sunday primes. |
| `keepalive` | `false` | Roll continuously instead, re-pinging `window_hours + margin_minutes` after each ping. Ignores the clock. |
| `prompt` | `ok` | What to send. Keep it trivial. |
| `model` | `haiku` | Cheapest model is fine; the window is account-level, not model-level. |

Times are local — Home Assistant passes its configured timezone into the
add-on, so no timezone option is needed.

## How the daily schedule behaves

The shipped defaults ping four times a day and then leave the small hours
alone:

```
08:00  ping   covers -> 13:00
13:05  ping   covers -> 18:05
18:10  ping   covers -> 23:10
23:15  ping   covers -> 04:15
       (no pings 04:15 - 08:00)
08:00  ping   covers -> 13:00      <- re-anchored
```

Note the five-minute creep. A prime scheduled exactly `window_hours` after
the previous one lands on the boundary, where it can be swallowed by the
still-open window and do nothing — costing a call and leaving a five-hour
gap. So when a prime falls at or before the estimated expiry, the add-on
defers it by `margin_minutes`. The drift accumulates through the day and
clears overnight, which is what keeps 08:00 identical every morning.

Keep the configured times as round numbers; the offsets are applied for you.
Spacing primes further apart than `window_hours` avoids the deferral
entirely, at the cost of gaps between windows.

`last_ping` is persisted to `/data`, so restarting the add-on resumes the
existing schedule rather than firing a fresh ping. If the add-on is down over
a prime and comes back with no window open, it pings once immediately to
catch up.

### Rolling mode

Set `keepalive: true` and `daily_primes: []` for continuous coverage with no
regard for wall-clock time: one ping at startup, then every
`window_hours + margin_minutes` forever. Costs roughly five windows a day,
which matters if your plan has a weekly cap.

## Entities

| Entity | Notes |
| --- | --- |
| `binary_sensor.claude_window_active` | On while the estimated window is open. |
| `sensor.claude_window_expires` | Timestamp; use with `relative_time` in a template. |
| `sensor.claude_window_remaining` | Minutes left, `0` when closed. |
| `sensor.claude_next_ping` | Timestamp, with a `reason` attribute. |
| `sensor.claude_keepalive_status` | `ok` / `error` / `never`, plus `last_ping` and `last_detail`. |

**These are estimates.** The add-on only knows about pings it made itself. If
you message Claude from your laptop at a moment when no window was open, you
started a window the add-on cannot see, and `window_expires` will be wrong
until the next ping. Check `/usage` inside Claude Code for the authoritative
numbers.

States are pushed through the Supervisor's Core API and re-published every
60 seconds, so they reappear on their own after a Home Assistant restart.

## Example automation

Nudge you if the morning prime failed:

```yaml
alias: Claude prime failed
triggers:
  - trigger: state
    entity_id: sensor.claude_keepalive_status
    to: error
actions:
  - action: notify.mobile_app
    data:
      title: Claude keepalive failed
      message: "{{ state_attr('sensor.claude_keepalive_status', 'last_detail') }}"
mode: single
```