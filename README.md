# Loxone API Documentation

Reverse-engineered API documentation extracted from the Loxone app bundle (`ftp://<loxone-miniserver>:21/web/commonv2.agz`) and the
[Lox AudioServer](https://github.com/lox-audioserver/lox-audioserver) TypeScript source (`src/`) which is a community project — not an
official Loxone document.

**⚠️ Not an official Loxone document!** Use at your own risk.

---

## Table of Contents

- [There are four distinct APIs](#there-are-four-distinct-apis)
- [API 1: Loxone App ↔ Loxone MiniServer](#api-1-loxone-app--loxone-miniserver-miniserver-apiyaml)
  - [Authentication flow](#authentication-flow)
  - [Control command format](#control-command-format)
  - [State update push (WebSocket)](#state-update-push-websocket)
  - [Selected control types and commands](#selected-control-types-and-commands)
- [APIs 2 & 3: AudioServer APIs](#apis-2--3-loxone-app--loxone-audioserver-and-loxone-miniserver--loxone-audioserver-openapiyaml)
  - [Shared commands (App + MiniServer)](#shared-commands-app--miniserver)
  - [MiniServer-only commands](#miniserver-only-commands)
  - [Response format](#response-format-both-clients)
  - [WebSocket events](#websocket-events-server--all-connected-clients)
- [API 4: Lox AudioServer Admin UI](#api-4-lox-audioserver-admin-ui--loxone-audioserver-audioserver-admin-apiyaml)
- [Viewing the docs](#viewing-the-docs)
- [Files](#files)
- [Sources](#sources)

---

## There are four distinct APIs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Loxone App  ──── (1) ────►  Loxone MiniServer                             │
│       │                           │                                         │
│       │ (2)                       │ (3)                                     │
│       ▼                           ▼                                         │
│   Loxone AudioServer  ◄──── (3) ──┘                                         │
│       │                                                                     │
│       │ (4)                                                                 │
│       ▼                                                                     │
│   Lox AudioServer Admin UI (browser)                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

| #   | Link                                           | Protocol                                    | Spec file                            |
| --- | ---------------------------------------------- | ------------------------------------------- | ------------------------------------ |
| 1   | **Loxone App ↔ Loxone MiniServer**             | WebSocket `jdev/sps/io/…`                   | `miniserver-api.yaml`                |
| 2   | **Loxone App ↔ Loxone AudioServer**            | HTTP GET + WebSocket `audio/…`              | `openapi.yaml` (App subset)          |
| 3   | **Loxone MiniServer ↔ Loxone AudioServer**     | HTTP GET + WebSocket `audio/…` + `secure/…` | `openapi.yaml` (full, superset of 2) |
| 4   | **Lox AudioServer Admin UI ↔ Lox AudioServer** | REST `/admin/api/…` with cookie auth        | `audioserver-admin-api.yaml`         |

**Key insight for APIs 2 and 3:** The App and the MiniServer use the **same `audio/…` command protocol**, but connect on **different ports**
and the MiniServer additionally sends a set of config-sync and pairing commands that the App never sends.

---

## API 1: Loxone App ↔ Loxone MiniServer (`miniserver-api.yaml`)

The Loxone App controls smart home devices by sending commands to the **Loxone MiniServer** over WebSocket or plain HTTP GET.

**WebSocket endpoint:** `ws://{miniserverHost}/ws/rfc6455` **HTTP mirror:** `GET http://{miniserverHost}/{command}`

### Authentication flow

```
1. GET /jdev/cfg/api            → discover firmware version & key algorithm
2. GET /jdev/cfg/getPublicKey   → fetch RSA-2048 public key
3. GET /jdev/sys/gettoken/{hash}/{user}/{permission}/{clientId}/{clientName}
                                → obtain auth token (hash = HMAC-SHA1 of credentials)
4. GET /jdev/sys/authwithtoken/{token}/{user}
                                → re-authenticate on reconnect
5. GET /jdev/sys/refreshtoken/{token}/{user}
                                → extend token before expiry
6. GET /jdev/sys/killtoken/{token}/{user}
                                → invalidate token on logout
```

### Control command format

```
jdev/sps/io/{uuid}/{command}
jdev/sps/io/{uuid}/{command}/{value}
```

`{uuid}` is the control's UUID from the structure file (`GET /data/LoxAPP3.json`).

### State update push (WebSocket)

The MiniServer pushes state changes as text frames:

```
{stateUuid}:{value}
```

State UUIDs are defined in `control.states` inside the structure file.

### Selected control types and commands

| Control type            | Commands                                                                   |
| ----------------------- | -------------------------------------------------------------------------- |
| `Jalousie` (blinds)     | Up, Down, Stop, FullUp, FullDown, Auto, shade/{pos}                        |
| `LightControllerV2`     | On, Off, plus, minus, setMoodToId/{id}, addMood, removeMood, learn         |
| `Dimmer`                | On, Off, setBrightness/{0-100}                                             |
| `ColorPickerV2`         | color/{hsv\|lxt}, setFavoriteColor/{index}, setSaturation/{val}            |
| `IRoomControllerV2`     | setOperatingMode/{0-5}, setComfortTemperature/{°C}, override/{mode}/{secs} |
| `Switch` / `Pushbutton` | On, Off, pulse, pulse/{durationMs}                                         |
| `Alarm`                 | arm, disarm/{code}, delayedOn/{secs}, acknowledge                          |
| `Gate` / `GarageDoor`   | open, close, stop                                                          |
| `DoorLock`              | lock, unlock                                                               |
| `Counter`               | reset, set/{value}                                                         |

---

## APIs 2 & 3: Loxone App ↔ Loxone AudioServer and Loxone MiniServer ↔ Loxone AudioServer (`openapi.yaml`)

The AudioServer runs **two HTTP + WebSocket servers** that share identical route handling. The App connects to one port; the MiniServer
connects to the other.

| Server    | Env var                  | Default port | Client            |
| --------- | ------------------------ | ------------ | ----------------- |
| `appHttp` | `LOXONE_APP_PORT`        | **7091**     | Loxone App        |
| `msHttp`  | `LOXONE_MINISERVER_PORT` | **7090**     | Loxone MiniServer |

The WebSocket **identification string** sent on connect differs:

```
App:         "LWSS V 16.1.10.01 | ~API:1.6~ | Session-Token: …"
MiniServer:  "MINISERVER V LWSS V 16.1.10.01 {mac} | ~API:1.6~ | Session-Token: …"
```

When the **MiniServer** WebSocket connects, the AudioServer also emits a **server heartbeat** with current zone state — the App does not
trigger this.

### Shared commands (App + MiniServer)

Both clients use these. All are HTTP GET; parameters are slash-separated.

```
# Zone control
GET /audio/{zoneId}/play
GET /audio/{zoneId}/pause
GET /audio/{zoneId}/volume/{0-100}
GET /audio/{zoneId}/shuffle/{enable|disable}
GET /audio/{zoneId}/repeat/{0|1|3}
GET /audio/{zoneId}/queueplus
GET /audio/{zoneId}/queueminus
GET /audio/{zoneId}/position/{seconds}
GET /audio/{zoneId}/status
GET /audio/{zoneId}/getqueue/{start}/{limit}
GET /audio/{zoneId}/serviceplay/{service}/{user}/{audiopath}
GET /audio/{zoneId}/roomfav/play/{slot}

# Config / management
GET /audio/cfg/getservices
GET /audio/cfg/getplaylists2
GET /audio/cfg/getroomfavs/{playerId}
GET /audio/cfg/search/{service}/{user}/{tag}/{query}/{limit}/{offset}
GET /audio/cfg/globalsearch
GET /audio/cfg/spotifyconnect/enable|disable
GET /audio/cfg/airplay/enable|disable
GET /audio/cfg/tunein/enable|disable
… (95+ shared endpoints total — see openapi.yaml)
```

### MiniServer-only commands

These are sent **only by the MiniServer** on its dedicated port — the App never sends them.

#### Pairing & Auth (`secure/…`)

Sent immediately after WebSocket connect as a bootstrap handshake:

| Command                    | Purpose                           |
| -------------------------- | --------------------------------- |
| `secure/info/pairing`      | Query paired MiniServer serial    |
| `secure/hello/{publicKey}` | Key-exchange bootstrap            |
| `secure/authenticate`      | Auth acknowledgement              |
| `secure/init`              | Session init (returns legacy JWT) |

#### Config Sync (`audio/cfg/…`)

Sent by the MiniServer to push or pull configuration state:

| Command                                | Purpose                                                     |
| -------------------------------------- | ----------------------------------------------------------- |
| `audio/cfg/ready`                      | MiniServer signals startup complete; gets session timestamp |
| `audio/cfg/getconfig`                  | Fetch config CRC-32 to detect stale cache                   |
| `audio/cfg/setconfig`                  | Push zone UUIDs and names to AudioServer                    |
| `audio/cfg/setconfigtimestamp`         | Sync config version timestamp                               |
| `audio/cfg/miniservertime`             | Sync wall-clock time                                        |
| `audio/cfg/volumes`                    | Bulk-set initial zone volumes                               |
| `audio/cfg/playername`                 | Set zone display name                                       |
| `audio/cfg/identify`                   | Send MiniServer serial + firmware version                   |
| `audio/cfg/defaultvolume/{zone}/{vol}` | Set startup volume                                          |
| `audio/cfg/maxvolume/{zone}/{vol}`     | Set volume cap                                              |
| `audio/cfg/eventvolumes/{zone}/{vol}`  | Set alert volume                                            |
| `audio/cfg/miniserverip`               | _(placeholder — ignored)_                                   |
| `audio/cfg/miniserverversion`          | _(placeholder — ignored)_                                   |
| `audio/cfg/timezone`                   | _(placeholder — ignored)_                                   |
| `audio/cfg/presencemode`               | _(placeholder — ignored)_                                   |
| `audio/cfg/groupopts`                  | _(placeholder — ignored)_                                   |
| `audio/cfg/speakertype`                | _(placeholder — ignored)_                                   |
| `audio/cfg/playeropts`                 | _(placeholder — ignored)_                                   |

### Response format (both clients)

```json
{ "{command}_result": <payload>, "command": "<original-path>" }
```

### WebSocket events (server → all connected clients)

| Event key                     | Trigger                 |
| ----------------------------- | ----------------------- |
| `audio_event`                 | Zone state changed      |
| `audio_queue_event`           | Queue contents changed  |
| `roomfavchanged_event`        | Room favorites changed  |
| `recentlyplayedchanged_event` | Recently played changed |
| `rescan_event`                | Library scan progress   |
| `globalsearch_result`         | Search result ready     |
| `audio_sync_event`            | Sync group changed      |
| `lineinchanged_event`         | Line-in inputs changed  |

---

## API 4: Lox AudioServer Admin UI ↔ Loxone AudioServer (`audioserver-admin-api.yaml`)

Internal REST API used by the AudioServer's browser-based Admin UI.

**Base URL:** `http://{host}:7091` **Auth:** Cookie session via `POST /admin/api/auth/login` (TTL 12 h)

| Group               | Base path                                                   | Notes                                                          |
| ------------------- | ----------------------------------------------------------- | -------------------------------------------------------------- |
| Auth                | `/admin/api/auth/…`                                         | login, logout, me                                              |
| Server info         | `/admin/api/info`                                           | public, no auth                                                |
| Zone admin states   | `/admin/api/zones/…`                                        | tech details, bulk purge                                       |
| Zone groups         | `/admin/api/groups`                                         | group listing                                                  |
| Transport discovery | `/admin/api/transports/…`                                   | AirPlay, Cast, DLNA, Sonos, Snapcast, Spotify, Music Assistant |
| Snapcast clients    | `/admin/api/snapcast/…`                                     | stream assignment                                              |
| Spotify             | `/admin/api/spotify/…`                                      | OAuth, librespot, bridges                                      |
| Apple Music         | `/admin/api/applemusic/…`                                   | Widevine key upload                                            |
| Line-In bridges     | `/api/linein/…`                                             | bridge register/status/ingest                                  |
| Audio streams       | `/streams/{zone}/{id}.{format}`                             | mp3/wav/pcm/aac/flac                                           |
| Media library       | `/admin/api/content/library/…`                              | upload, rescan, storages                                       |
| Custom radio        | `/admin/api/content/radio/…`                                | add/delete/validate                                            |
| Logs                | `/admin/api/logs/…`                                         | list, SSE stream, level control                                |
| Config              | `/admin/api/config/…`                                       | CRUD for server config                                         |
| Alert sounds        | `/admin/api/alerts/…`                                       | upload, list, revert                                           |
| Updates             | `/admin/api/adminui/update`, `/admin/api/components/update` |                                                                |

---

## Viewing the docs

**Online:** Visit the [GitHub Pages Swagger UI](https://mr-manuel.github.io/Loxone_api_documentation/) to browse the docs without any local setup.

**Locally:** Open `index.html` in a browser. It loads the YAML specs locally via [Swagger UI](https://swagger.io/tools/swagger-ui/) (fetched from
unpkg.com — requires internet).

The landing page has **three tabs**:

| Tab                                    | Spec file                    | Covers              |
| -------------------------------------- | ---------------------------- | ------------------- |
| **MiniServer API**                     | `miniserver-api.yaml`        | API 1               |
| **AudioServer – App & MiniServer API** | `openapi.yaml`               | APIs 2 & 3 combined |
| **AudioServer – Admin API**            | `audioserver-admin-api.yaml` | API 4               |

In the **App & MiniServer API** tab, routes are grouped by tag:

- Tags **without a prefix** → used by both App and MiniServer
- Tags starting with **`MiniServer ›`** → MiniServer only

---

## Files

```
api-docs/
├── index.html                    # Swagger UI with 3-tab spec switcher
├── openapi.yaml                  # APIs 2 & 3: App + MiniServer audio protocol
├── miniserver-api.yaml           # API 1: MiniServer smart home commands
├── audioserver-admin-api.yaml    # API 4: AudioServer Admin REST API
└── README.md                     # This file
```

## Sources

| File / directory                                                                                          | Used for                                                      |
| --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `ftp://<loxone-miniserver>:21/web/commonv2.agz`                                                           | MiniServer control types, `audio/…` schemas, WebSocket events |
| `https://github.com/lox-audioserver/lox-audioserver/src/adapters/loxone/commands/router/routeRegistry.ts` | Exact route list for both App and MiniServer                  |
| `https://github.com/lox-audioserver/lox-audioserver/src/adapters/loxone/http/loxoneHttpService.ts`        | Two-port server architecture, heartbeat behaviour             |
| `https://github.com/lox-audioserver/lox-audioserver/src/config/loxone.ts`                                 | Port names (`appHttp`/`msHttp`), identification strings       |
| `https://github.com/lox-audioserver/lox-audioserver/src/adapters/loxone/commands/handlers/`               | Handler logic, MiniServer-only config sync                    |
| `https://github.com/lox-audioserver/lox-audioserver/src/adapters/http/adminApi/adminApiHandler.ts`        | Admin API routes                                              |
| `https://github.com/lox-audioserver/lox-audioserver/src/adapters/http/streams/`                           | Audio stream and proxy endpoints                              |
| `https://github.com/lox-audioserver/lox-audioserver/src/adapters/http/lineInApi/`                         | Line-In bridge API                                            |
