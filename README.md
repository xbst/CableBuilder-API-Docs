# CableBuilder API Docs

**Generate links that open [https://cable.isiks.tech/](https://cable.isiks.tech/) with a cable already configured.**

A share URL carries a complete harness configuration — connectors, every wire, pin
assignments, twisted pairs, length. Opening one loads that exact cable into the
configurator, where the customer can review it, edit it, add it to the cart and check
out. This doc covers everything needed to generate valid URLs from another tool.

Three things to know up front:

1. **URLs never carry prices.** The site prices every cable server-side from its own
   catalogue; nothing in a URL can change what is charged.
2. **Out-of-stock parts don't break a link.** The cable still loads exactly as
   configured, a banner lists what is unavailable, and ordering is blocked until the
   customer swaps the part or it is restocked.
3. **A bad link fails safely.** An unreadable URL shows an error and the default
   configurator; it never half-loads a different cable.

---

## Quick start

A 500 mm cable, JST XH 4-pin on both ends, four 24 AWG silicone wires, straight-through:

```
https://cable.isiks.tech/?cable=1&len=500&a=XH 4-Pin&b=XH 4-Pin&w=1,B1,24,Silicone,Red&w=2,B2,24,Silicone,Black&w=3,B3,24,Silicone,Blue&w=4,B4,24,Silicone,Green
```

Percent-encode when your HTTP library requires it (space → `%20`, `>` → `%3E`); the
site accepts both encoded and literal forms.

---

## The two formats

| Format | Marker | Use it when |
|---|---|---|
| **Long form** | `?cable=1&…` | Recommended for integrations: human-readable, hand-writable, debuggable in the address bar. |
| **Short form** | `?c=<base64url>` | When URL length matters (chat apps, QR codes). Same data, compact. |

Both decode identically. Everything below describes the long form first; the short
form is a compact spelling of the same fields.

---

## Discovering valid parts

Three public, same-origin GET endpoints return live catalogue data. Query them rather
than hardcoding parts — stock, gauges and bounds change.

### `GET /api/connectors`

Every connector currently orderable (active and in stock). Relevant fields:

```jsonc
{
  "id": "cmqwkxv7o000xs63ublbbmxnw",   // stable — preferred reference for generated URLs
  "name": "XH 4-Pin",                  // usable as a reference too (may be renamed)
  "pinCount": 4,
  "rows": 1, "cols": 4,                // pin layout; see "Pin numbering" below
  "family": {
    "name": "JST-XH",
    "compatibleGauges": "[22,24,26,28]",  // JSON-encoded array, as a string
    "doubleCrimpGauges": "[]"             // gauges allowed 2-per-pin; empty = never
  }
}
```

### `GET /api/wire-options`

Every wire type. Relevant fields:

```jsonc
{
  "gauge": 24,
  "material": "Silicone",              // must match exactly, case-sensitive
  "colorStock": [ { "color": "Red", "stock": 30000 }, ... ]
}
```

### `GET /api/labor-pricing`

Bounds your generator must respect (current values shown; read them live):

```jsonc
{
  "minLengthMm": 50,  "maxLengthMm": 2000,   // cable length; out-of-range is clamped on load
  "minStripMm": 2,    "maxStripMm": 10       // strip length on bare ends; 0 = cut, not stripped
}
```

---

## Long form reference

The presence of `cable` marks a share URL. All other parameters are optional unless
noted. Unknown parameters are ignored, so you may add your own (e.g. `utm_source`).

| Parameter | Required | Value |
|---|---|---|
| `cable` | yes | Format version. Always `1` for now. |
| `len` | no | Cable length in mm. Omitted → 300. Clamped to the labor-pricing bounds. |
| `a` | yes | Connector at end A (the source end) — see *Connector references*. |
| `b` | yes | Connector at end B. |
| `cc` | no | Third connector (C), making a breakout/Y harness. |
| `split` | with `cc` | How many wires go to C. Set it to the number of `w` entries whose destination is `C`. |
| `ob`, `oc` | no | Pin order for the B / C run: `straight` (default) or `reversed`. Ignored when `map=1`. |
| `map` | no | `1` = the pin assignments in `w` are exact and must be kept as written. Set it whenever your wiring is anything other than the plain straight/reversed pattern (double crimps, skipped pins, custom order). |
| `sa`, `sb`, `sc` | no | Strip length in mm for a *bare* end A / B / C. `0` = cut flush. Otherwise must be within the strip bounds. |
| `w` | yes, repeated | One wire per parameter — see *Wire syntax*. At least one required. |
| `tp` | no, repeated | One twisted pair per parameter — see *Twisted pairs*. |

### Wire syntax

Each `w` parameter is five comma-separated fields:

```
w = srcPin , destination , gauge , material , colour

srcPin       pin number on connector A (1-based)
destination  B or C followed by the pin number, e.g. B3 or C1 (case-insensitive)
gauge        AWG number, e.g. 24
material     exactly as returned by /api/wire-options (case-sensitive), e.g. Silicone
colour       colour name (case-insensitive) or 0-based palette index
```

Example: `w=3,B1,20,PTFE,Red` — wire from A pin 3 to B pin 1, 20 AWG PTFE, red.
A material containing a comma is fine: the colour is always the last field.

### Colour palette

Fixed, indexed 0–7. Availability per gauge/material comes from `colorStock`.

| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|
| Black | Red | Blue | Green | Yellow | White | Orange | Brown |

### Connector references

`a`, `b` and `cc` accept any of:

- **Connector id** (from `/api/connectors`) — stable across renames; **use this for
  machine-generated URLs**.
- **Connector name**, matched case-insensitively (`xh 4-pin` works) — better for
  hand-written links; breaks if the connector is renamed.
- **`bare`** — no connector: the wires simply end there, optionally stripped
  (`sa`/`sb`/`sc`). Any end may be bare; both ends bare is a plain cut of wire.

A connector that is out of stock still resolves (see behavior above). A connector
that was removed from the catalogue makes the link fail with a named error.

### Pin numbering

Pins are numbered 1‥`pinCount`, running across each row: on a dual-row connector
with `rows: 2, cols: 3`, row one is pins 1–3 and row two is pins 4–6. This matches
the diagrams the site renders.

### Twisted pairs

Each `tp` parameter names the two wires of one pair by **wire key** — `A<srcPin>>B<dstPin>`
(or `C<dstPin>`), the two keys separated by a comma:

```
tp=A1>B2,A4>B1
```

Wires are named by *both* endpoints because with double crimping a pin alone is
ambiguous. Rules:

- Both keys must exactly match a `w` entry.
- Both wires must end on the same connector (B with B, C with C).
- A wire may belong to at most one pair.

Twisting is priced in: each pair adds a per-pair labor charge and its wires are
billed slightly longer (the helix overhead, currently 10%).

---

## Rules the site enforces

A URL that breaks these still loads whatever it can, but the customer sees an error
and cannot order until it is fixed — so generate within them:

| Rule | Detail |
|---|---|
| Pins must exist | Every `srcPin` ≤ A's `pinCount`; every destination pin ≤ that connector's `pinCount`. Out-of-range mappings get auto-renumbered, which changes the cable — don't rely on it. |
| Max 2 wires per pin | And only when the connector family's `doubleCrimpGauges` contains **both** wires' gauges. A family with `[]` never doubles. |
| Gauge compatibility | A wire's gauge must be in the `compatibleGauges` of connector A's family **and** of its destination connector's family. Bare ends impose no restriction. |
| Material | Exact, case-sensitive match against `/api/wire-options`. |
| Length | Clamped to `minLengthMm`‥`maxLengthMm`. |
| Strip | `0`, or within `minStripMm`‥`maxStripMm`. Only meaningful on bare ends. |
| Wire count | ≥ 1; effectively bounded by connector A's pins (× 2 where doubling is allowed). |

---

## Short form

The same configuration as one `c` parameter: a JSON object, UTF-8 encoded, base64url
encoded (`+`→`-`, `/`→`_`, padding `=` stripped).

| Key | Long-form equivalent | Notes |
|---|---|---|
| `v` | `cable` | Version, `1`. |
| `l` | `len` | Length in mm. |
| `a`, `b`, `c` | `a`, `b`, `cc` | Connector references. Omit `c` for two-ended cables. |
| `x` | `split` | Wires to C. Only with `c`. |
| `ob`, `oc` | `ob=reversed`, `oc=reversed` | `1` when reversed; omit when straight. |
| `m` | `map=1` | `1` when the mapping is exact. |
| `sa`, `sb`, `sc` | same | Strip lengths; omit when 0. |
| `t` | `tp` | Array of pairs: `[["A1>B2","A4>B1"], …]`. |
| `w` | `w` | Array of wire strings, same 5-field syntax. Colour by index is shortest. |

The quick-start cable as a short form — this JSON:

```json
{"v":1,"l":500,"a":"cmqwkxv7o000xs63ublbbmxnw","b":"cmqwkxv7o000xs63ublbbmxnw",
 "w":["1,B1,24,Silicone,1","2,B2,24,Silicone,0","3,B3,24,Silicone,2","4,B4,24,Silicone,3"]}
```

becomes:

```
https://cable.isiks.tech/?c=eyJ2IjoxLCJsIjo1MDAsImEiOiJjbXF3a3h2N28wMDB4czYzdWJsYmJteG53IiwiYiI6ImNtcXdreHY3bzAwMHhzNjN1YmxiYm14bnciLCJ3IjpbIjEsQjEsMjQsU2lsaWNvbmUsMSIsIjIsQjIsMjQsU2lsaWNvbmUsMCIsIjMsQjMsMjQsU2lsaWNvbmUsMiIsIjQsQjQsMjQsU2lsaWNvbmUsMyJdfQ
```

---

## Versioning

- Both forms carry a version (`cable=1` / `"v":1`). Generate version 1.
- Unknown parameters and JSON fields are **ignored** — links keep working as the
  site gains features.
- A link with a version **higher** than the site understands is refused with a clear
  message rather than misread. If the format ever moves to version 2, version-1
  links keep working.

---

## Worked examples

All verified to load and price on the live site.

**Reversed 2-pin** (connector B physically flipped, so A1→B2, A2→B1):

```
/?cable=1&len=200&a=XH 2-Pin&b=XH 2-Pin&ob=reversed&w=1,B2,26,Silicone,Red&w=2,B1,26,Silicone,Black
```

**Breakout** — Micro-Fit 2x2 trunk fanning out to a data connector (24 AWG) and a
power connector (20 AWG). Note `split=2` matches the two `C` wires:

```
/?cable=1&len=300&a=Micro-Fit 2x2&b=XH 2-Pin&cc=Micro-Fit 2-Pin&split=2&w=1,B1,24,Silicone,White&w=2,B2,24,Silicone,Blue&w=3,C1,20,Silicone,Red&w=4,C2,20,Silicone,Black
```

**CAN harness** — six wires between two Micro-Fit 2x3, data lines double-crimped two
per pin (Micro-Fit allows 24/26 AWG doubling), two twisted pairs, exact mapping
(`map=1`):

```
/?cable=1&len=800&a=Micro-Fit 2x3&b=Micro-Fit 2x3&map=1&w=1,B2,24,PTFE,Green&w=2,B4,20,PTFE,Black&w=3,B2,24,PTFE,Green&w=4,B1,24,PTFE,Yellow&w=5,B3,20,PTFE,Red&w=6,B1,24,PTFE,Yellow&tp=A1>B2,A4>B1&tp=A3>B2,A6>B1
```

**Bare end** — XH 3-pin to bare wires: cut at 150 mm, stripped 5 mm:

```
/?cable=1&len=150&a=XH 3-Pin&b=bare&sb=5&w=1,B1,24,Silicone,Red&w=2,B2,24,Silicone,Black&w=3,B3,24,Silicone,White
```

---

## Generator checklist

For a tool (or an LLM) building a URL from a wiring requirement:

1. `GET /api/connectors`, `/api/wire-options`, `/api/labor-pricing`.
2. Pick connectors with enough pins; reference them **by id**.
3. For every wire, choose a gauge in **both** end families' `compatibleGauges`, a
   material/colour combination with stock, and assign pins within range.
4. Putting two wires on one pin? Check the family's `doubleCrimpGauges` contains
   both gauges — and set `map=1`.
5. Using a third connector? Set `cc` and `split` = number of `C` wires.
6. Twisted pairs: keys must match `w` entries exactly; same destination; one pair
   per wire.
7. Clamp `len` to the labor-pricing bounds yourself so the customer sees the length
   you intended.
8. Percent-encode the query string.

Questions or a part you need listed: contact the shop before working around the
catalogue — anything not returned by the APIs can't be ordered anyway.
