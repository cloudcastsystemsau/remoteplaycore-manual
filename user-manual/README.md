# Remote Play user manual

Remote Play is a web gateway for **Axia Livewire** and **AES67** audio-over-IP.
It listens to what is advertising on the broadcast LAN, presents every source
in one browser console, and transcodes any of them to Opus so an operator can
monitor them from a laptop, a phone, or a wall-mounted panel — without a
hardware receiver and without joining the multicast groups themselves.

This manual covers the operator and administrator console. It is written
against the shipping build; every label in **bold** is quoted verbatim from the
UI.

> **This revision is not yet complete.** It covers signing in, the Sources
> console, source lists, monitoring panels, audio outputs, server settings and
> licensing. Device management, federation and the REST surface are named where
> they appear but not yet documented in full — see
> [What this revision does not cover](#what-this-revision-does-not-cover).

## Contents

1. [Core concepts](#1-core-concepts)
2. [Getting started](#2-getting-started)
3. [Sources](#3-sources)
4. [Source lists](#4-source-lists)
   — [the list editor](#the-list-editor), [parent nodes](#parent-nodes), [renaming an entry](#renaming-an-entry)
5. [Monitoring panels](#5-monitoring-panels)
   — [assigning sources](#assigning-sources), [pad labels and lamp colours](#pad-labels-and-lamp-colours), [panel settings](#panel-settings), [visibility](#visibility-personal-private-and-public), [the kiosk view](#the-kiosk-view)
6. [Audio outputs](#6-audio-outputs)
   — [configuring an AES67 output](#configuring-an-aes67-output), [routing sources](#routing-sources-to-outputs)
7. [Server administration](#7-server-administration)
   — [Groups](#groups), [General](#general), [TLS](#tls), [Email](#email), [Backups](#backups)
8. [Licence](#8-licence)
9. [What this revision does not cover](#what-this-revision-does-not-cover)

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
| **Admin** | **Summary**, **Users**, **Groups**, **API Keys**, **Legacy Servers**, **Audio Outputs**, **Federation** |
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

**New list** creates one, and **View** opens the editor.

### The list editor

![The list editor](img/20-listeditor.png)

The editor is two panes. **Available sources** on the left is the live
inventory, grouped by device with a **Search sources…** box; **List contents**
on the right is the list being built. A source row's **+** adds it (*"Add to
list"*); one already used shows a tick instead (*"Already in the list"*). Rows
can also be dragged across — *"Drag sources here, or use a row's + button."*

The list name and its entry count sit above both panes, and edits are staged
until **Save changes** — an **Unsaved changes** marker appears meanwhile, and
leaving asks *"Discard unsaved changes?"*. **Go back** returns to the list
index.

#### Parent nodes

**Add parent node** creates a named group inside the list — *Morning Show* and
*Utility* in the capture above. Entries that belong to no group collect under
**Ungrouped**, and each node shows how many entries it holds. A node can be
renamed, and removing one is not destructive: *"Remove parent node (entries
stay, ungrouped)"*. An empty node prompts *"Drag sources onto this node."*

Parent nodes are how a long list stays readable — a studio's list can separate
the show feeds from the utility ones without needing two lists.

#### Renaming an entry

The pencil on any entry sets a **display name** for that entry only; the
underlying source is untouched, and the same source in another list keeps its
own name. A renamed entry carries a **renamed → {original}** badge, so nobody
has to guess what *Brekky Host* or *Spare Feed* actually points at. The device
each entry came from is shown on the right of its row.

This is the answer to terse plant labels: the console can say *Brekky Host*
where the device advertises *GUEST MIC 1*.

#### Ordering and removal

Each entry has a drag handle plus **Move up** / **Move down** arrows, and the
red **×** is **Remove from list**. Order is the list's own — it is what the
Sources tree and the panel drawer show when filtered to that list.

#### Sources that are not on air

Because entries are stored by their stable `origin:type:channel` key, a list
keeps working across restarts and re-advertisements. An entry with no live
match is marked **missing** rather than dropped: *"No live source matches this
entry — it is kept but unplayable."* Filtering the Sources tree to a list whose
entries are all off air says so plainly: *"None of this list's entries match a
live source right now."*

A source that never advertises at all can still be listed. **Add manual
source** takes it directly — by Livewire channel, AES67 multicast address or
soundcard — and the **add by SDP or multicast address** flow parses a pasted
SDP blob or an address and port, storing the result as a `manual` source.

#### Who can see a list

A list is visible to every signed-in user (*"Every signed-in user sees this
list."*) or restricted to named accounts and groups: **Allowed users** and
**Allowed groups**, where *"Only the ticked users and groups (plus admins) see
this list."* With no groups defined the picker says *"No groups defined —
create them under Admin › Groups."* See [Groups](#groups).

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
panel as *"Already on the panel"*.

Clicking an empty **ASSIGN** pad is the quicker route — it opens a picker for
that slot alone, titled **Assign source to slot {n}**:

![Assigning a source to a pad](img/24-buttonsourceassignment.png)

The picker is the same inventory, grouped by device with protocol badges and a
**Search sources…** box, scoped to the one slot you clicked. Choosing a source
fills the pad and closes the picker.

### Pad labels and lamp colours

Clicking a pad that already has a source opens its editor.

![The pad editor](img/23-buttoneditor.png)

**BUTTON LABEL** overrides what the pad shows — *"Custom label (blank = source
name)"* — and takes effect on **Apply**. **LAMP COLOUR** picks the pad's lamp
from **Amber**, **Red**, **Green**, **Blue**, **Violet** or **Pink**, which is
what makes a wall panel readable from across a studio: mics on one colour,
returns on another. **Clear slot** empties the pad without touching the layout
around it.

A pad whose source has left the inventory keeps its label and shows **MISSING**
beneath it, so the layout never reshuffles when a device drops off air.

### Layouts

Panels are saved as named layouts. **Layouts** lists them, **Save as…** stores
the current arrangement under a new name, and **Manage layouts…** renames and
deletes. The active layout is marked **active**, and unsaved changes show an
**unsaved** chip beside the panel name. Layouts are per-user: yours are yours
unless an admin publishes a panel as a room panel.

### Panel settings

**SETTINGS** on the panel toolbar opens **Panel settings**, where an
administrator turns a personal layout into a room panel.

![Panel settings](img/15-panel-settings.png)

| Setting | What it does |
|---|---|
| **VISIBILITY** | **PERSONAL**, **PRIVATE** or **PUBLIC** — see below. |
| **RACK** | **1RU** or **2RU**. 1RU is a single row of pads; 2RU is two. |
| **PANEL OUTPUT** | The audio output the active pad's source is routed to. Room panels require one: *"Assign an output — room panels route the active pad's source to it."* |
| **Stream audio to the browser** | *"On: the loaded view also monitors the source. Off: a silent control surface that only switches the physical output."* |
| **PUBLIC PATH** | The slug a public panel answers on, e.g. `/p/studio-1`. Must be unique. |
| **Allowed users** | Private panels only: *"Comma-separated usernames who may access this private panel. Blank = any logged-in user."* |
| **Allowed groups** | Private panels only: *"Members of a ticked group may access this private panel (in addition to the users above)."* With no groups defined it reads *"No groups defined — create them under Admin › Groups."* |

The allow-lists appear only when **PRIVATE** is selected — a public panel is
open to every session, and a personal one to nobody but its owner.

**COLS** on the toolbar sets how many pads per row, independently of the rack
size. A 1RU panel with four columns is four pads; a 2RU panel with eight is
sixteen.

Once an output is bound, the panel carries a banner reading *"Pads switch the
source feeding {output}"*, with a **CONTROL ONLY** chip when browser audio is
off. That is the wall-panel case: the operator presses a pad, the physical
output follows, and nothing is streamed to the browser at all.

### Visibility: personal, private and public

| Visibility | Who reaches it | Output binding |
|---|---|---|
| **PERSONAL** | Only its owner. *"Your own monitoring layout — browser listen only, no output binding."* | None — a personal panel cannot drive an output. |
| **PRIVATE** | Any signed-in user; or, once **Allowed users** or **Allowed groups** is set, only those accounts and the members of those groups (admins always). | Required. |
| **PUBLIC** | Any signed-in user, at the panel's own path `/p/{slug}`. | Required. |

> **Public does not yet mean anonymous.** The in-app hint reads *"reachable at
> /p/{slug} with no login (the kiosk view lands in wave 2)"*. The kiosk view
> **has** shipped, but the no-login half has not: the endpoint behind it is
> still gated on a valid user session, and an external panel token — the `pt_`
> credential issued under **Panel control** — satisfies only the panel policy,
> not this one. A wall display therefore still needs a signed-in browser
> session today. Treat the hint as stale on both counts.

### The kiosk view

A public panel opens at `/p/{slug}` as a chrome-free view: the panel name, the
rack, and nothing else — no sidebar, no navigation away from it.

![The kiosk view of a public panel](img/16-kiosk-public-panel.png)

Pads behave exactly as they do in the console: pressing one makes it the active
pad and routes its source to the panel's output. A pad whose source is not in
live inventory shows **MISSING** rather than disappearing, so a wall panel does
not silently reshuffle when a device drops off air. If the slug does not match
a public panel, the view says so: *"No public panel at "{slug}" — check the
address with your administrator."*

---

## 6. Audio outputs

An audio output makes the server an audio **emitter**, not just a browser
gateway. A source routed to an output is passed through as live audio — RTP in,
PCM out — *in addition to* the Opus stream a browser might be listening to.
This is what lets a panel press change what comes out of a studio's overhead
speakers.

Outputs need GStreamer on the server. **Audio Outputs** is admin-only.

![Audio outputs](img/12-audio-outputs.png)

The table lists each output's **NAME**, **KIND**, **STATUS** (**running**,
**stopped** or **error**), an **ENABLED** toggle, **FORMAT** and **ENDPOINT**.
**Test** pushes tone to the output so it can be confirmed at the speaker
without routing a source to it.

Three kinds:

| Kind | Endpoint | Notes |
|---|---|---|
| **ALSA** | A device string such as `hw:1,0` | Linux hosts. |
| **AES67** | A multicast group, port and packet time | Sends AES67 onto the AoIP network. |
| **ASIO** | An ASIO driver name | *"ASIO runs on Windows hosts only."* On a non-Windows console the picker is disabled with *"This console isn't running on Windows — the ASIO device can't be configured from here."* |

### Configuring an AES67 output

![Editing an AES67 output](img/13-audio-output-aes67.png)

| Field | Meaning |
|---|---|
| **Name** | Free text; what the panel's output picker and the routing editor show. |
| **Kind** | **AES67**. |
| **Multicast address** | The group the stream is sent to, e.g. `239.70.9.1`. Validated: *"Enter a valid multicast address (e.g. 239.69.1.5)"*. |
| **port** | Destination UDP port; AES67 convention is 5004. |
| **packet time** | Packet time in **ms**. 1 ms is the AES67 interop point — at 48 kHz that is 48 frames per packet. |
| **Adapter** | The egress interface. **Default route (OS chooses)** lets the host decide; on a multi-homed server pick the AoIP NIC explicitly, or the RTP leaves on the wrong network. An adapter that no longer exists is marked **not found on this host**. |
| **Channels** / **Sample rate** | 2 ch @ 48000 for a normal stereo AES67 stream. |
| **Enabled** | Whether the sender runs. |

The output is announced over SAP so AES67 receivers can discover it, and the
announcement declares the sender's clock. A clock chip shows which: **PTP**
(*"Locked to PTP grandmaster … — clock-compatible with the AES67 plant"*) or
**Free-run** (*"Free-running on the system clock. Receivers that require a
shared PTP clock may refuse to subscribe."*). Free-run is legal and announced
honestly, but a Dante or Ravenna receiver may decline it.

### Routing sources to outputs

**SOURCE ROUTING**, below the table, is the persistent assignment: *"Assign the
sources that feed each output. Search by source name, device, channel or
multicast address — a source can feed only one output."*

Each output lists what feeds it, and carries a **Panel: {name}** chip when a
panel drives it — *"This output is driven by a panel — the selected pad routes
its source here."* A routed source that has left the inventory is marked
**offline**: *"This source is routed but is not in the live inventory right now
— the device may be offline or renamed. The route is kept."* Routes survive the
source going away, which is what you want when a studio is simply switched off
overnight.

Changes are staged until **Save routing**; an **Unsaved routing changes** note
appears while they are.

## 7. Server administration

### Groups

A group collects users so a panel or a source list can be granted to all of
them at once, rather than naming each account on every panel.

![Groups](img/19-groups.png)

**New group** creates one; **Members** picks the accounts in it. The page's own
summary states the rule: *"Groups collect users so a panel or source list can be
granted to all of them at once. Access is granted to a user directly or through
any group they belong to."* Deleting a group is not silent about consequences —
*"Delete group "{name}"? Panels and lists that grant it lose that grant."*

Grants are additive and independent: a user reaches a private panel if they are
named on it directly **or** belong to any group it grants. Administrators always
have access regardless.

> The **Allowed groups** hint in panel settings says *"Enforced in wave 2"*.
> For private panels that is out of date — group membership is already checked
> server-side when deciding who may drive a panel.

### General

![General settings](img/10-settings-general.png)

**Public base URL** is the address the server advertises for itself; blank
derives it from the request. **OIDC SSO** and **LDAP directory sign-in** are
configured here, each with **Role mappings** that translate a claim value or
directory group to the **admin** or **user** role — *first match wins*, with a
**Default role** and an option to **refuse unmapped users**. The redirect URI
to register with an OIDC provider is shown for copying, and *"Internal sign-in
always remains available as break-glass"*, so a misconfigured provider cannot
lock everyone out. **Test connection** validates LDAP against the **saved**
settings, so save before testing.

**Streaming / Opus** controls the encoder every browser listener gets:
**Complexity** (*"0 = lightest CPU, 10 = best quality"*), **Application**
(**Audio (music)**, **VoIP (speech)** or **Low delay**), **Variable bitrate**
and **Force stereo**.

### TLS

![TLS settings](img/17-settings-tls.png)

**Current certificate** shows the installed certificate's **Subject**,
**Issuer**, **Validity**, **Thumbprint** and **Subject alt names**, along with
its source (**uploaded** or **self-signed**) and how long it has left — *"Valid
— {days} days remaining"*, *"Expires in {days} days … renew soon"*, or
*"Expired on {date}"*.

Below it, **HTTPS binding** enables HTTPS on a chosen port (*"Install a
certificate before enabling HTTPS"*), **Generate self-signed certificate**
mints one for a list of DNS names or IPs, and **Upload PFX / PKCS#12** installs
a real certificate. **Remove certificate** disables HTTPS again.

### Email

![Email settings](img/18-settings-email.png)

*"Outgoing mail server used for notifications such as backup-failure alerts."*
Server, port, SSL/TLS, optional credentials and the From address. A stored
password is never returned to the browser — the field shows **password set**,
and *"leave the field empty to keep it"*.

### Backups

**Settings › Backups** is a placeholder in this build: *"Scheduled device
config backups are coming with a later stage of the port."* Device
configuration backup itself lives under **Devices**.

## 8. Licence

![The Licence page](img/11-licence.png)

**Settings › Licence** shows whether the installation is **Licensed** or
**Unlicensed**, and for a licensed one the **Licensed to** name, **Serial**,
**Issued** date and **Expires** date — which may read **Never (perpetual)**.

Activation takes a serial: enter it under **Activate licence** and press
**Activate**. Activation is bound to the machine, and the page shows the
**Hardware ID:** the licence is issued against, which is what support needs
when a serial has to be reissued for new hardware. **Deactivate licence**
releases it.

A licence grants named features. Three are gated in this build:

| Feature | What it unlocks |
|---|---|
| **Federation** | Aggregating sources from peer servers. |
| **External Stream Deck / API** | Panel tokens and the external control API. |
| **Device firmware & backups** | Firmware banks and device configuration backup/restore. |

Reaching a gated feature without a licence shows the reason inline: *""{feature}"
requires a licence — install one under Settings › Licence."* Audio monitoring
itself is not gated — an unlicensed server still discovers sources and streams
them to a browser, so a plant is never taken off air by a licence problem.

---

## What this revision does not cover

These areas exist in the product and are reachable from the sidebar, but are
not yet documented here. They are the subject of the next revisions:

- **Devices** — Axia device management: LWRP terminal, configuration backup and
  restore, scheduled backup tasks, firmware banks.
- **Federation** and **Legacy Servers** — aggregating sources from peer
  RemotePlay servers and from legacy RemotePlay installations.
- **Summary** and **Users** — the admin overview and account management.
- **Panel control** — issuing `pt_` panel tokens for external controllers.
- The REST API and WebSocket streaming contract.

**API Keys** and **Settings › Backups** are placeholders in this build and say
so in the UI; there is nothing to document yet.

---

*Generated against main @ `5294269`, RPC-1…RPC-115. Screenshots captured with
`build/manual-screenshots.mjs` against a live instance fed by
`RemotePlayCore.LivewireSim` and `RemotePlayCore.Aes67Sim`; see
[manual-maintenance.md](../manual-maintenance.md).*
