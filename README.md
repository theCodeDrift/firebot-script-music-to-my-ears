# Music to My Ears

A custom script for [Firebot](https://github.com/crowbartools/Firebot) that lets your Twitch
viewers request music and control playback through **Spotify**. It adds a small set of Firebot
**effects** (Actions) that you wire into your own commands, channel-point redemptions, or buttons.
You pick the trigger and write the chat messages. The script handles search, moderation, queueing,
and playback.

> Spotify Premium is required. Queue, skip, play, and pause all need a Premium account with an
> active playback device (the Spotify app open and playing somewhere).

## Features

- **Request Song** searches Spotify, moderates the result, and queues it in one atomic action. It
  searches tracks only, so podcasts and episodes never come back. It outputs a single `success` flag plus an
  `errorReason`, so you can branch on one condition (to refund a redemption, for example).
  - A **blocklist** of terms, artists, and tracks, set per effect and globally. Terms are pre-filled
    with `karaoke`, `instrumental`, and `inst.`
  - An **allow explicit tracks** toggle per effect, off by default
  - **Jukebox protection**: cooldowns on the same track, on an artist for everyone, and on an
    artist per viewer, so one band can't take over the queue. There's also a **maximum track
    length**.
  - A ready-to-send **`errorText`** for every failure, and an **`errorCooldown`** telling the viewer
    when to try again
- **Skip Track**, **Play / Resume**, and **Pause** are single-action playback controls.
- **Get Current Track** outputs the now-playing track plus **who requested it**.

The script doesn't register a system command. The trigger (`!sr`, a reward, a button) and every
chat reply are yours to design from the effects' outputs.

## Requirements

- Firebot v5 (5.65+)
- A Spotify **Premium** account
- A Spotify application (free to create) for OAuth credentials

## Installation

1. **Create a Spotify app** at the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
   - Add this **Redirect URI**, exactly as written. Spotify won't accept `localhost`, so it has to
     be `127.0.0.1`:
     ```
     http://127.0.0.1:7472/api/v1/auth/callback
     ```
   - Copy the **Client ID** and **Client Secret**.
2. **Download** `musicToMyEars.js` from the [latest release](../../releases/latest).
3. In Firebot, go to **Settings → Scripts → Manage Startup Scripts → Add New Script** and select
   the downloaded file.
4. Open the script's settings, paste your **Client ID** and **Client Secret**, and save. Then
   **fully restart Firebot**, since the credentials are read when the integration registers at
   startup.
5. Go to **Settings → Integrations**, find **Music to My Ears (Spotify)**, and **Link** your
   account. Authorize in the browser and you'll be redirected back, with the integration showing as
   connected.

## Configuration (script settings page)

| Setting                             | Description                                                                                                                                                                                                         |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Spotify Client ID / Secret**      | From your Spotify app. Changing these requires a Firebot restart.                                                                                                                                                   |
| **Country of Play**                 | Optional two-letter country code (`US`, `GB`, `DE`). Requests resolve against that country's catalog, and region-locked tracks are rejected as `not-playable`. Leave it blank to use your linked account's country. |
| **Blocked artists, tracks & terms** | A global blocklist applied to every Request Song effect, ahead of each effect's own list. Same format as the per-effect list (see [Blocklist](#blocklist)).                                                         |
| **Copy debug log**                  | Copies this session's log to your clipboard for a bug report.                                                                                                                                                       |

The other moderation settings and the restrictions below are set on each Request Song effect, so a
clean `!sr` and a looser one can differ.

### Blocklist

Both blocklists take one entry per line.

- A Spotify **artist** or **track** link, URI, or bare id blocks that exact artist or track. This
  lets you ban an artist without banning their name as a word, and it catches featured artists as
  well as the primary credit.
- Any other line is a **term**, matched case-insensitively against the artist and track names.

A match fails the request with `blocked-artist`, `blocked-track`, or `blocked-term`. Blocks are
checked before explicit content, track length, and cooldowns, since a blocked track will never
succeed.

### Restrictions (per Request Song effect)

| Restriction                              | Default | Description                                                           |
| ---------------------------------------- | ------- | --------------------------------------------------------------------- |
| **Same track cooldown (minutes)**        | `30`    | How long before the same track can be requested again.                |
| **Artist cooldown (minutes)**            | `0`     | How long before that artist can be requested again **by anyone**.     |
| **Artist cooldown per viewer (minutes)** | `0`     | How long before the **same viewer** can request that artist again.    |
| **Maximum track length (seconds)**       | `0`     | Longest track that may be queued. `420` is a reasonable 7-minute cap. |

Set any restriction to `0` to turn it off. Only the same-track cooldown is on by default. Turn on
the artist cooldown once one band starts crowding the queue.

The artist cooldown applies to **every viewer**. It's there to keep the playlist varied, so a viewer
who hasn't requested anything may still be told an artist played recently. That's intended. Only
the track's **primary** artist is gated.

When more than one cooldown applies at once, the effect reports the one with the **longest**
remaining wait. That way `errorCooldown` is always a time when the request will actually succeed.

### Reporting a problem

The script keeps its own log of everything it does during the Firebot session: every search, every
candidate Spotify returned, and the reason behind every refused request. It logs all of that
whatever your Firebot log level is, so there's nothing to turn on first.

When a request goes wrong, open the script settings page, click **Copy debug log**, and paste the
result into your bug report. It includes the script and Firebot versions and your settings, with the
client id and secret left out. The log lives in memory only, so copy it before restarting Firebot.

## Usage

Wire the effects into your own command. A typical song-request command (`!sr`):

1. **Command trigger** `!sr` (or a channel-point reward).
2. **Request Song (Spotify)** effect:
   - **Search query**: `$arg[all]` (or `$redemptionMessage` for a reward). It also accepts a
     Spotify track link (or `spotify:track:` URI or id), which queues that exact track. If the
     command trigger leaks into the text (`!sr Toxic by Britney Spears`, from a raw message or a
     viewer who repeats the trigger), it's stripped, so you can wire the whole message straight in.
   - Adjust the blocklist and explicit toggle to taste
3. **Branch on the outputs** (read effect outputs with `$effectOutput[name]`):
   - When `$effectOutput[success]` is `true`, add a **Chat** effect:
     `🎵 Queued "$effectOutput[trackName]" by $effectOutput[artistName]`
   - Otherwise, send `$effectOutput[errorText]` straight to chat, or branch on
     `$effectOutput[errorReason]` for your own wording (refund the redemption, whisper the viewer,
     and so on)

### Request Song outputs

| Output                                  | Meaning                                                                                                                                                                                                                                                     |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `success`                               | `true` if the track passed moderation and was queued                                                                                                                                                                                                        |
| `trackUri` / `trackName` / `artistName` | The matched track (empty on failure). On `blocked-artist`, `artistName` is the artist that was blocked, even when it's a featured credit.                                                                                                                    |
| `errorReason`                           | `not-found` · `not-playable` · `blocked-term` · `blocked-artist` · `blocked-track` · `explicit` · `too-long` · `recently-played` · `artist-recently-played` · `user-artist-recently-played` · `no-active-device` · `not-premium` · `not-linked` · `unknown` |
| `errorText`                             | A friendly explanation of the failure, ready to send to chat (empty on success)                                                                                                                                                                             |
| `errorCooldown`                         | Seconds until the reported cooldown expires. `0` when the failure wasn't a cooldown.                                                                                                                                                                        |

### Get Current Track outputs

`isPlaying`, `trackName`, `artistName`, `trackUri`, `requestedBy` (who requested the current track
through Request Song, or empty if it wasn't a request), plus `errorReason` and `errorText`.

The playback effects (**Skip / Play / Resume / Pause**) each output `success`, an `errorReason`
(`no-active-device`, `not-premium`, or `not-linked`), and an `errorText` so you can react to
failures.

## Development

```bash
pnpm install
pnpm build           # production single-file bundle → dist/musicToMyEars.js
pnpm build:dev       # dev build + deploy into Firebot (see below)
pnpm test            # jest
pnpm lint            # eslint
pnpm typecheck       # tsc --noEmit
```

To auto-deploy a dev build into your Firebot profile, point `FIREBOT_SCRIPTS_DIR` at your profile's
`scripts` folder:

```bash
FIREBOT_SCRIPTS_DIR="/path/to/Firebot/v5/profiles/Main Profile/scripts" pnpm build:dev
```

Architecture and conventions are in [.conventions/architecture.md](.conventions/architecture.md).
The project uses [OpenSpec](https://github.com/Fission-AI/OpenSpec) for spec-driven development.
Run it through the package script (`pnpm openspec <command>`).

### Releasing

Versioning and changelogs are managed with [Changesets](https://github.com/changesets/changesets):

1. Add a changeset describing your change: `pnpm changeset`
2. On merge to `main`, CI opens a **"Version Packages"** PR that bumps the version and updates the
   changelog.
3. Merging that PR builds the bundle, tags the version, and publishes a **GitHub Release** with
   `musicToMyEars.js` attached.

## License

MIT
