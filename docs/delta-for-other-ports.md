# Joining Delta: for other ports and server software

For people who build their own Halo CE port, client or dedicated server
(OpenH1, say) and want their games on the Delta network: listed on
stats.chupathingy.org and in Chupathingy's Online Games, with carnage reports
and stats, and players who can join from Chupathingy.

Delta is open ([delta.md](delta.md), "An open network"). It doesn't depend on
being Chupathingy, and nothing about it is secret except the signing keys.
This page sorts it into steps. Each step works without the ones after it.
The protocol details are in [delta.md](delta.md), and the rules for which
builds play together are in [delta-compatibility.md](delta-compatibility.md).

## The ground rule

OpenCE's game protocol is the common ground. Whatever your own netcode does,
Delta players can join your servers only through it: OpenCE's advertisement,
join and game messages, with the OpenCE legacy number. A server that speaks
only your own protocol can still be listed (step 1). The list will show it
as what it is, and Chupathingy players won't be offered a join they can't
make.

## Step 1: get listed (Delta List)

Needs: nothing from us but the URL. Your server announces itself over HTTPS.

- `POST https://stats.chupathingy.org/v1/announce`, form encoded, every
  few seconds while the game is up. `POST /v1/withdraw` when it stops.
- Fields: `invite`, `name`, `map`, `engine`, `players`, `maximum_players`,
  `open`, `score_limit`, `teams`, `version` (the OpenCE legacy number the
  server announces), then the host fields, then `roster` last. The reference
  is `announce()` and `host_fields()` in `port/linux/src/browser.c`. The host
  fields are in [delta.md](delta.md), "The host's fields".
- `invite` is the OpenCE invite (`halo://join/…`). It's how a player's copy
  reaches your server through the invite tunnel, even behind NAT. A server
  without one can be listed, but nobody can join it from the list.
- Announce `delta=0` until you speak Delta Peer (step 3). The site then lists
  your games with protocol `opence`.
- Send your own `build` (your version string), so the list can tell your
  servers apart from ours.
- One listing per game. Please be kind to the site.

Reading the list: `GET /v1/games.txt?fields=23`. Each field is described in
[delta.md](delta.md), "Delta List", and in
[playlist-maps.md](playlist-maps.md).

## Step 2: carnage reports and stats (Delta Stats)

Needs: step 1. A token from the site's operator (Milenko) if you want your
games to count as a trusted server's.

- End of game: `POST /v1/report` (the carnage report), as `browser.c` sends
  it.
- The event stream: `POST /v1/events`, JSON, gzip. The format is schema 2,
  in [delta.md](delta.md), "Delta Stats" ("The batch", "Map facts", "The
  upload").
- Without a token, the site takes a game only while the server lists it,
  and only from the same address. With one (`HALO_EVENTS_TOKEN` on our
  server), its games count as a trusted server's.
- Never send a hardware ID or an address. Players are their names.

## Step 3: speak Delta Peer

Needs: steps 1 and 2 aren't required. A platform number from us if your
port runs somewhere new.

Delta Peer is a UDP port beside the game's (host on 5160, the game's 5150
plus 10), carried through the same invite tunnel as the game's datagrams.
It's what turns on platform icons, profiles, chat, server messages,
moderation and Halo PC map identity between your machines and ours.

1. **Advertise it.** Put the Delta tag at the tail of your advertisement's
   reserved bytes (`DELTA_ADVERTISED_TAG_OFFSET`, [delta.md](delta.md),
   "The advertisement"). Put it there only while your Delta socket is open.
   OpenCE machines ignore it.
2. **Handshake.** HELLO / WELCOME / LEGACY, BYE, every datagram a 12-byte
   header and at most 1200 bytes ([delta.md](delta.md), "Handshake", "Wire
   format"). No answer within 4 seconds means plain OpenCE for that game, so
   a mistake on either side degrades rather than breaks.
3. **Platform key.** 8 bytes in HELLO and WELCOME ("Platform keys"). If your
   build runs on a platform already in the registry (Windows, Linux...),
   use its number. For a new platform, ask for a number first: open an issue
   or ask on the Discord ("Platform numbers"). Don't pick one yourself.
4. **Capabilities.** Offer only the bits you actually implement
   ("Capability registry"). Unknown bits are ignored both ways. A
   reasonable first set for a dedicated server: `platform`,
   `server_messages`, `chat`, `moderation`.
5. **Announce it.** Once Delta Peer is up, announce `delta` (the major),
   `platform_key`, `capabilities` and `machines` in step 1's fields.

Test against our dedicated server and a Chupathingy client before shipping.
The parsers on our side treat every message as hostile, and yours should
too ("Security").

## Step 4: stay compatible as OpenCE moves

Delta games go by our wire ID, and OpenCE games go by OpenCE's legacy
number ([delta.md](delta.md), "The legacy number"). For a server speaking
OpenCE's protocol:

- **Announce one legacy number,** the one your build really plays. An OpenCE
  client joins only on an exact match.
- **Follow the signed table if you want to keep up without a release.**
  `GET /v1/delta/legacy` and `/v1/delta/legacy.sig`, Ed25519, newest
  serial wins ("The legacy table as config"). It says which numbers each of
  our wires announces and joins. A row for your own build can be added only
  once the cross-play gate has played it against ours and OpenCE's
  ("Following OpenCE").
- **Never send message 92** ("Never send message 92").

### Getting into the cross-play gate

The gate (`tools/crossplay_test.py`) plays real games between release builds:
host and client both ways, on an Xbox map, spawns, deaths, shots, score, the
next game, a join in progress and a rejoin. To add your build to it we need:

- a way to build or download a Linux release of your server and client at a
  given tag, with no window (or headless flags), on the NTSC Xbox maps;
- the hooks the test drives (start a playlist, join an address, log joins,
  spawns, deaths and the end of a game), or a description of how your logs
  say those;
- your OpenCE legacy number per release, and whether your OpenCE-protocol
  code changes when your own netcode does.

## The other way: our clients on your servers

A server whose own netcode is better than OpenCE's doesn't have to give it
up to play with us. Chupathingy clients can learn to join it, the way they
learned OpenCE's network versions: added beside the OpenCE protocol, gated,
with OpenCE as the fallback ([delta-compatibility.md](delta-compatibility.md),
principles 4 and 6). For that we need to read your server's side: its source,
or a written spec of how it advertises, how a client joins and what the game
messages are. Then the list gets a `protocol` value for it, and our clients
join it where they can.

## What we need from you

To wire a new port or server in, send us:

1. **What it is.** A client, a dedicated server or both, and the platforms
   it runs on (for platform numbers, step 3).
2. **Its OpenCE level.** The OpenCE build or legacy number its OpenCE
   protocol matches, and how you plan to follow OpenCE's raises.
3. **Maps.** Which families it hosts (Xbox, Custom Edition, HaloMD, Halo PC
   retail) and how it identifies a Custom Edition map (the game record's map
   version: a header checksum, or 29's hash prefix).
4. **Invites.** Whether it uses OpenCE's invite tunnel and brokers, or its
   own addressing (then a player's copy can't reach it from our list yet:
   tell us how you'd want it reached).
5. **Your own protocol.** Whether you want your own netcode listed alongside
   (a `protocol` value of its own on the list), so your clients can find
   your servers on our list too.
6. **Your server's code or spec,** if you want Chupathingy clients to join
   your own netcode, not only your OpenCE-protocol servers (above).
7. **Contact.** Who to talk to when the gate fails against your build.
