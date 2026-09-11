# BILRESA E2490 — Zigbee mode, captured

What the remote actually puts on the air in its undocumented Zigbee mode.
Captured against an ESP32-C6 Zigbee coordinator; see the main
[README](README.md) for how to enter the mode, and for the Matter side.

> IKEA does not support this mode, and a remote living in it receives **no
> firmware updates** — those only arrive over Matter. Zigbee2MQTT notes the
> Zigbee stack is stripped to essentials and could be removed in a future OTA.
> There is precedent: IKEA closed DIRIGERA's OpenThread API, officially a
> "debugging interface", once people started building on it.

The mode itself was first found and published by u/Remarkable-Loquat-38 in
[BILRESA on Zigbee](https://www.reddit.com/r/tradfri/comments/1plqavn/bilresa_on_zigbee/)
(r/tradfri, 13 December 2025), on ZHA. Most of what the wider community knows
about this mode traces back to that thread.

## Device identity

| | |
|---|---|
| Zigbee model id | **`09BA`** (the E2489 dual-button sibling is **`09B9`**) |
| Manufacturer | IKEA of Sweden, manufacturer id **4476** |
| Hardware version | 2 (both models) |
| Type | Battery-powered Zigbee End Device (sleepy) |
| Firmware seen in interviews | 1.8.5, date code 20250226 — same build as shipped |
| E2489 date code | 20250206 |

Manufacturer id 4476 is the same number as the Matter vendor id 0x117C, which
is a convenient cross-protocol confirmation that you are looking at one device
wearing two stacks.

Those model ids are what Z2M and ZHA match on, and the fastest way to tell from
a log which BILRESA you are looking at.

## Endpoint and cluster map

The remote presents **one endpoint (1)**:

| Direction | Clusters |
|---|---|
| Input (server) | `genBasic`, `genPowerCfg`, `genIdentify`, `genPollCtrl` |
| Output (client) | `genIdentify`, `genGroups`, `genScenes`, `genOnOff`, `genLevelCtrl`, `touchlink` |

The distinction matters and is easy to get backwards. On/Off and Level are
**output** clusters: the remote is a *client* that sends commands to a group. It
does not host an `OnOff` or `CurrentLevel` attribute for a coordinator to read
or subscribe to — everything below is a command stream, not attribute reporting.

Note also that this is a flat, single-endpoint device. The three product groups
are **not** separate endpoints here, unlike the nine endpoints Matter exposes.

## Only standard ZCL

There are **no manufacturer-specific commands at all**, which makes the remote
far easier to integrate than IKEA's older STYRBAR/TRÅDFRI remotes.

## Scroll wheel — `MoveToLevel` on Level Control `0x0008`

A continuous dimmer that **streams absolute level values** as repeated
`MoveToLevel` commands — not deltas, not `Step`, not `Move`. One command every
**101 ms** while turning.

Captured, turning down then reversing back up:

```
120 112 104  96  88  79  70  61  52   <- turning down
 55  58  61  64  67  70  73  76  79   <- reversed, turning up
 83  87  91  95  99
```

Two behaviours worth handling:

- **Velocity sensitive.** Step size tracks how fast you turn: ~8–9 per tick
  when fast, ~3 per tick when slow.
- **Direction reversals break the cadence.** Gaps of 18 ms and 58 ms appeared
  exactly where the direction changed, against the otherwise steady 101 ms.

The command payload is malformed by ZCL's own rules, and Zigbee2MQTT flags it:
missing option mask and override parameters, a level of **255** where the ZCL
maximum for lights is 254, and a **transition time of 1 second** — nonsensical
for a live dimmer emitting a new target every 101 ms, and the likely reason
some integrations feel sluggish. Some integrations surface that 255 as **null**
instead. Parse leniently, clamp, and handle null.

Zigbee2MQTT's device page carries the same warning: "The MoveToLevel brightness
command lacks parameters and may exceed optimal brightness levels." Its own
workaround for the null-at-maximum case shipped in
[zigbee-herdsman-converters#11244](https://github.com/Koenkk/zigbee-herdsman-converters/pull/11244),
the PR that added scroll-wheel support.

Values can also arrive **out of order** on some coordinators — reported
independently by more than one person on the Reddit thread, not reproduced in
the captures here.

## Wheel click — `On` / `Off` on On/Off `0x0006`

The click sends alternating **On** and **Off** commands, not `Toggle`. State is
kept per group on the remote, which keeps group members in sync with each other
but drifts if you also control the group from elsewhere.

**A click is preceded by a Level-to-zero**, then the level is restored ~100 ms
later:

```
t=16053  Level  0x0008  value=0    <- level slammed to zero
t=16054  On/Off 0x0006  value=1    <- 1 ms later, the actual press
t=16162  Level  0x0008  value=99   <- 108 ms later, level restored
```

Any code that maps Level straight onto an output will visibly glitch on every
single click. Either ignore a Level of 0 that is immediately followed by an
On/Off, or debounce Level by ~150 ms.

## The group button is invisible to a coordinator

Pressing the button below the LEDs changes which group the remote drives, but
**emits nothing to a coordinator**. A coordinator therefore sees one flat action
stream and cannot tell the three groups apart. In practice this is what stops
the community HA blueprints from driving more than one target at a time — the
Matter integration can, this one cannot.

This is a limit of the *coordinator* path, not of Zigbee mode as such. A device
bound by **Touchlink** does see the channel, because the remote groupcasts to
21658, 21659 or 21660 according to the selected channel, and group membership
determines delivery. Verified on hardware 2026-09-10, M5NanoC6, ESP-IDF 5.4.2.

Recovering the group ID is awkward, because the esp-zigbee-sdk does not expose
it. For a groupcast, `zb_zcl_parsed_hdr_t.addr_data.common_data.dst_addr` reads
0xfffd, since an APS groupcast travels inside a network-layer broadcast, and
`esp_zb_apsde_data_ind_t` reports `dst_addr_mode=0x02` with
`dst_short_addr=0xfffd`, the group having been resolved away. The APS frame
control still marks the frame as group-delivered: bits 2–3 of `fc`, where 3
means group.

The way to get the channel back is to give each group its own endpoint, 21658
on endpoint 1, 21659 on endpoint 2 and 21660 on endpoint 3, then read
`dst_endpoint`. Note that binding a channel makes the remote send Groups Remove
All Groups followed by Groups Add Group to endpoint 1, which destroys that
layout unless it is re-asserted afterwards.

Working implementation in
[esp32c6-zigbee-probe](https://github.com/tdamsma/esp32c6-zigbee-probe).

## Battery reporting is not trustworthy

Battery reporting is configured with a **minimum interval of 3600 minutes and a
maximum of 65000** — so a freshly joined device may say nothing useful for a
long time.

Worse, the value itself is routinely mishandled. A published interview shows
**"112 % remaining"**: ZCL `BatteryPercentageRemaining` is in **half percent**
(0–200), so 224 raw becomes 112 when something forgets to halve it. This is the
same encoding trap as the Matter side (see README §3.7), and it is why several
people report an obviously wrong battery figure in ZHA. Treat any battery
reading over 100 as a doubled value. Fixed upstream in
[zigbee-herdsman-converters#11997](https://github.com/Koenkk/zigbee-herdsman-converters/pull/11997).

## Other exposed attributes

`battery`, `voltage`, `identify`, `action`. The action enum differs between the
two models, because the wheel and the buttons emit different Level Control
commands:

| Model | Action enum |
|---|---|
| E2490 scroll wheel | `on`, `off`, `on_double`, `off_double`, `brightness_move_to_level` |
| E2489 dual button | `on`, `off`, `on_double`, `off_double`, `brightness_move_up`, `brightness_move_down`, `brightness_stop` |

Double clicks send IKEA's custom Scenes command `TradfriArrowSingle` (command
`0x07`) with param1 `256` on the on-side or `257` on the off-side, against group
65289 — the same command IKEA's older remotes use to step WS bulbs to the next
or previous scene.

Zigbee2MQTT decodes this directly into `on_double` and `off_double` through a
dedicated helper, `ikeaBilresaDouble()` (`src/lib/ikea.ts:745`). The command
definitions themselves (`tradfriArrowSingle` 0x07, `tradfriArrowHold` 0x08,
`tradfriArrowRelease` 0x09) are in `addIkeaGenScenesCluster()` in the same file.
A bulb that does not understand `TradfriArrowSingle` simply ignores the frame,
which does not stop Z2M from acting on it.

## Integration support

Both devices were unsupported at discovery and appeared as a generic, unnamed
Zigbee device. Support has since landed in both stacks.

| Stack | State |
|---|---|
| Zigbee2MQTT | Supported. Both models handled by [zigbee-herdsman-converters#11154](https://github.com/Koenkk/zigbee-herdsman-converters/pull/11154), raised from issues [#30321](https://github.com/Koenkk/zigbee2mqtt/issues/30321) (E2490) and [#30325](https://github.com/Koenkk/zigbee2mqtt/issues/30325) (E2489). Recognised from zigbee-herdsman-converters **25.98.0** / Z2M **2.7.2**. |
| ZHA | **Partial.** Quirk v2 in [zha-device-handlers#4612](https://github.com/zigpy/zha-device-handlers/pull/4612) (`zhaquirks/ikea/bilresa2btn.py`), merged **24 March 2026**, covers `09B9` (E2489) only. There is no quirk for the `09BA` scroll wheel; how its triggers should surface is tracked in [#4991](https://github.com/zigpy/zha-device-handlers/issues/4991), and the original device-support request is [#4647](https://github.com/zigpy/zha-device-handlers/issues/4647). |

Before the converter landed, the auto-generated Z2M definition exposed only the
action endpoint — battery plus basic on/off and level — with the wheel and the
remaining buttons dead. That half-working state is what most of the early
reports describe, and it is worth checking your versions before concluding the
hardware is at fault.

The ZHA quirk is worth reading even if you are not using ZHA, because of what it
had to work around: it adds a custom `IkeaBilresaLevelControl` cluster that
**tracks the Move direction internally**, because the release event does not
carry the direction it was moving in. It also replaces the Scenes cluster, and
exposes eight device-automation triggers — short press on/off, long press dim
up/down, long release dim up/down, double press up/down. Reviewer MattWestb's
notes there also cover the battery-percentage handling above.

Note that double-press availability is an **integration** property, not a
device one: ZHA users saw no double-click or long-press events at all until the
quirk existed, while Z2M exposed `on_double` / `off_double` from the converter
onwards. Hold is a genuine difference between the two models, though — the
E2489's buttons emit `Move`/`Stop`, which integrations surface as hold; the
E2490 has no equivalent, and no long press exists on the Matter side at all
(README §3.4).

## Coordinator-dependent failure — unresolved

Several people pair successfully and then receive **no button events at all**,
left with battery and link quality only. The common thread is the radio:
**EFR32MG24 and EFR32MG21** coordinators (the ZBT-2 named explicitly). Tracked
as [bellows#708](https://github.com/zigpy/bellows/issues/708).

Treat the correlation as **unconfirmed**. The issue is open, has no assigned
owner, and the reporter says plainly that they suspect "something in bellows
that is blocking or not generating the `zha_event` on the event bus" but cannot
verify it without access to affected hardware — their own Sonoff Dongle P
(firmware 20211217) is unaffected. The dataset is small. If you see this
symptom, the radio is the first thing to try changing; the captures in this
document were taken on an ESP32-C6 coordinator and never showed it.

## Other quirks

- **Sleepy end device.** It will not answer coordinator queries while asleep —
  press a button to wake it.
- **Pairing is unreliable**, beyond being timing sensitive. Repeated interview
  failures, several attempts needed, and at least one report of the device
  appearing in the coordinator minutes later with no visible confirmation. The
  LEDs going dark is the only signal you get, so give it time before retrying.
- **Low click sensitivity.** Light presses click audibly without registering.
  Press firmly.
- **~0.5 s built-in delay** on clicks, to disambiguate multi-press. There is no
  `initial_press` event, unlike SOMRIG. (The Matter side *does* emit
  InitialPress — see the main README.)
- **Curiosity:** once in Zigbee mode, the remote can be added to the IKEA Home
  Smart app, where it shows up as a **SYMFONISK remote**.
