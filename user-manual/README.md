# Remote Play user manual

Remote Play is a web gateway for **Axia Livewire** and **AES67** audio-over-IP.
It listens to what is advertising on the broadcast LAN, presents every source
in one browser console, and transcodes any of them to Opus so an operator can
monitor them from a laptop, a phone, or a wall-mounted panel — without a
hardware receiver and without joining the multicast groups themselves.

This manual covers the operator and administrator console. It is written
against the shipping build; every label in **bold** is quoted verbatim from the
UI.

> **This revision is a first pass.** It covers signing in, the Sources console,
> source lists and monitoring panels. Devices, audio outputs, federation,
> licensing and the REST surface are named where they appear but not yet
> documented in full — see [What this revision does not cover](#what-this-revision-does-not-cover).

## Contents

1. [Core concepts](#1-core-concepts)
2. [Getting started](#2-getting-started)
3. [Sources](#3-sources)
4. [Source lists](#4-source-lists)
5. [Monitoring panels](#5-monitoring-panels)
6. [What this revision does not cover](#what-this-revision-does-not-cover)

---

## 1. Core concepts

**Sources are discovered, not configured.** Remote Play joins the Livewire
advertisement group (239.192.255.3:4001) and the SAP groups AES67 senders
announce on (224.2.127.254 and 239.255.255.255, both port 9875). Everything
advertising on the network it can see appears in the console within a few
seconds. Nothing is typed in by hand unless a source does not advertise at all,
in which case it can be added manually from an SDP or a multicast address.

**A source keeps its identity across restarts.** Every source has a stable key
of the form `origin:type:channel` — origin is `local` for something this server
discovered itself, `manual` for a hand-added source, or the id of a legacy
RemotePlay server it was aggregated from; type is `lwto`, `lwfrom`, `aes67` or
`soundcard`. Lists and panels store that key and re-resolve it against live
inventory, so a source that goes off air and comes back returns to the same
list entry and the same panel pad rather than becoming a new one.

**Two roles, not four.** Every account is either **admin** or **user**. Admins
see the whole console, including the **Admin** and **Settings** groups in the
sidebar. Users see **Sources**, **Panel** and **Source Lists**, and their own
saved panel layouts. Older installations that carried operator or viewer roles
coerce them to user on upgrade.

**Monitoring is Opus over WebSocket.** Pressing **Play** on a source opens a
WebSocket to the server, which joins the multicast group on the operator's
behalf, buffers the RTP, and encodes to Opus at the bitrate chosen in the
player. The operator's browser never touches the AoIP network.

## 2. Getting started

### Requirements

| | |
|---|---|
| Server OS | Linux or Windows, .NET 8 runtime |
| Network | An interface on the broadcast LAN, able to join multicast groups. On a multi-homed host, name it with `--Livewire:BindAddress=<ip>`. |
| Browser | Any current Chromium, Firefox or Safari. Opus playback needs no plugin. |
| Audio outputs (optional) | GStreamer 1.20 or later on the server, for passing audio through to a physical or network output. |

### First sign-in

A fresh installation with no user accounts creates a bootstrap administrator
and logs a warning saying so. Sign in with **admin** / **admin**.

![Sign-in page](img/01-sign-in.png)

The sign-in page states this directly: *"First boot creates a default admin
account when no users exist — change its password after signing in."* Do that
immediately — **Password** at the bottom of the sidebar. If the server is
configured for LDAP or an OIDC provider, directory accounts sign in through the
same two fields, and any configured SSO provider appears as its own
**Sign in with …** button.

### The console at a glance

![The console after signing in](img/02-console-at-a-glance.png)

The sidebar is the whole navigation model. The Remote Play mark at the top is
the home affordance — clicking it returns to **Sources**. Below it:

| Group | Entries |
|---|---|
| *(top level)* | **Sources** (with **All Sources**, **Local Sources** and one entry per source list), **Panel**, **Source Lists**, **Devices** |
| **Admin** | **Summary**, **Users**, **API Keys**, **Legacy Servers**, **Audio Outputs**, **Federation** |
| **Settings** | **General**, **TLS**, **Email**, **Backups**, **Licence** |

The **Admin** and **Settings** groups are admin-only. At the bottom sit the
signed-in account, a light/dark toggle, and **Password** / **Sign out**.

## 3. Sources

**Sources** is the console's home page: every device the server can see,
grouped by device, with each device's source count on the right. The header
counts what has been found, and a device row expands to its sources.

Each source row carries its channel number, name and multicast address, a
protocol badge — **LW** for Livewire, **AES67** for an AES67 stream — and a
**Play** button. Device rows carry the badge too, so a plant running both
protocols can be read at a glance.

### Finding a source

![Searching the Sources tree](img/03-sources-tree.png)

**Search devices and sources…** filters the tree as you type, matching both
device and source names — above, `Studio 1` has narrowed a plant of a hundred
sources to two devices. Three toggles control how the tree is laid out:

| Toggle | Options | Effect |
|---|---|---|
| **DEVICES** | **Name** / **IP** | Label device rows by name or by IP address |
| **SOURCES** | **Name** / **Mcast** | Label source rows by name or by multicast group |
| **SORT** | **Name** / **IP** | Order the tree alphabetically or by address |

**Expand all** and **Collapse all** act on the whole tree. On a plant with
thousands of sources, collapse first and search — the tree is built for that
scale.

### The source filters

Under **Sources** in the sidebar:

- **All Sources** — everything, including sources aggregated from federated
  peers and legacy RemotePlay servers.
- **Local Sources** — only what this server discovered itself.
- **One entry per source list** — see [§4](#4-source-lists). Lists you can see
  but not edit carry a padlock.

### Listening

**Play** on a source row starts monitoring it; **Stop** ends it. The player
carries a level meter and a bitrate selector. Some rows cannot be played and
say why:

| State | Meaning |
|---|---|
| **locked** | *"Source is locked on the device"* — the device itself will not release it. |
| **remote** | The source belongs to a federated peer; *"streams via origin server"*. |
| *manual* | *"Manual sources can't be monitored from this page yet."* |

## 4. Source lists

A source list is a curated set of sources — a studio's regular feeds, an
engineer's fault-finding set, the sources a particular panel draws from. Lists
appear as their own entries under **Sources**, so choosing one filters the tree
to its members.

![Source Lists](img/04-source-lists.png)

The page separates **My lists** from **Shared lists**; a shared list you do not
own is marked **Shared** and **read-only**. The table shows **Name**,
**Entries**, **Visibility** and **Updated**. Visibility is either **All users**
or an explicit set (**1 user** / **{n} users**).

**New list** creates one. Within a list you can add entries from live inventory,
group them under parent nodes, and give any entry a display name that differs
from the source's advertised name — useful where a plant's channel labels are
terse. Because entries are stored by their stable key, a list keeps working
across restarts and re-advertisements; entries whose source is not currently on
air simply do not resolve, and the filtered tree says so: *"None of this list's
entries match a live source right now."*

A source that does not advertise at all can still be listed: **add by SDP or
multicast address** parses a pasted SDP blob, or takes a multicast address and
port directly, and stores the result as a `manual` source.

## 5. Monitoring panels

A panel is a rack-unit-styled grid of pads. Each pad is bound to a source;
pressing it monitors that source, and — on an admin-configured room panel —
routes it to the panel's audio output.

![The monitoring panel](img/05-panel.png)

The panel toolbar carries, left to right: the panel name and **PRESET**
selector, **COLS** with − / + to set the pad columns, a **RACK** toggle between
**1RU** and **2RU**, the visibility chip (**PERSONAL**), **SETTINGS**, and
**SOURCES** — which opens the source drawer on the right.

Underneath sits the player strip: state (**IDLE** until a pad is pressed), a
stereo **METER**, a **BITRATE** selector and **Stop**. An unassigned pad reads
**ASSIGN**.

### Assigning sources

![The source drawer](img/06-panel-source-drawer.png)

**SOURCES** opens the drawer: *"Drag a source onto a panel slot to assign it.
Locked sources can't be assigned."* The drawer has its own **Filter** (any
source list) and **Search sources…** box, and marks sources already on the
panel as *"Already on the panel"*. Clicking an empty pad opens the same picker
for that slot directly.

Each pad can carry a **Button label** — *"Custom label (blank = source name)"* —
and a **Lamp colour**, so a wall panel can be read at a glance from across a
studio.

### Layouts

Panels are saved as named layouts. **Layouts** lists them, **Save as…** stores
the current arrangement under a new name, and **Manage layouts…** renames and
deletes. The active layout is marked **active**, and unsaved changes show an
**unsaved** chip beside the panel name. Layouts are per-user: yours are yours
unless an admin publishes a panel as a room panel.

### Room panels

An administrator can promote a panel from **personal** to a private or public
room panel, bind it to an audio output, and give it a slug. A public panel then
opens at `/p/{slug}` as a kiosk view with no sign-in, scoped to exactly that
panel's sources and output — the wall-panel case. Selecting a pad on a room
panel drives the bound output; deselecting stops it.

---

## What this revision does not cover

These areas exist in the product and are reachable from the sidebar, but are
not yet documented here. They are the subject of the next revisions:

- **Devices** — Axia device management: LWRP terminal, configuration backup and
  restore, scheduled backup tasks, firmware banks.
- **Audio Outputs** — server-side passthrough to ALSA/ASIO devices and AES67
  streams via GStreamer, and the IO editor.
- **Federation** and **Legacy Servers** — aggregating sources from peer
  RemotePlay servers and from legacy RemotePlay installations.
- **Summary**, **Users**, **API Keys** — administration.
- **Settings** — **General**, **TLS**, **Email**, **Backups**.
- **Licence** — activation, feature gates and the unlicensed preview state.
- The REST API and WebSocket streaming contract.

---

*Generated against main @ `HEAD`, RPC-1…RPC-102. Screenshots captured with
`build/manual-screenshots.mjs` against a live instance fed by
`RemotePlayCore.LivewireSim` and `RemotePlayCore.Aes67Sim`; see
[manual-maintenance.md](../manual-maintenance.md).*
