# Media Stack — Radarr, Sonarr, Prowlarr, qBittorrent, FlareSolverr

Automated movie/TV acquisition feeding the [Jellyfin media server](media-server.md):
Prowlarr aggregates indexers, Radarr/Sonarr decide what to grab, qBittorrent
downloads it, FlareSolverr solves Cloudflare challenges Prowlarr's HTTP client
can't handle on its own. All five run as Docker containers on the
[Docker VM](docker-vm.md). Added 2026-08-2x.

## Services

| Service | Job | Internal port |
|---|---|---|
| Prowlarr | Indexer aggregator — one indexer config shared with Radarr/Sonarr | 9696 |
| Radarr | Movie collection manager — searches, grabs, renames/imports | 7878 |
| Sonarr | Same as Radarr, for TV | 8989 |
| qBittorrent | Torrent client, actually downloads | 8080 |
| FlareSolverr | Headless-browser proxy that solves Cloudflare challenges for Prowlarr | 8191 (not published — Prowlarr reaches it by container name only) |

## Access

Tailscale only, same pattern as every other admin surface in this homelab
(see [Lessons Learned](lessons-learned.md) — this stack originally shipped
LAN-wide and got locked down after a security audit, same story as
Homepage and Pi-hole before it):

```yaml
qbittorrent:
  ports:
    - "127.0.0.1:8081:8080"
    - "6881:6881"       # torrent peer traffic — has to stay open
    - "6881:6881/udp"
prowlarr:
  ports:
    - "127.0.0.1:9696:9696"
radarr:
  ports:
    - "127.0.0.1:7878:7878"
sonarr:
  ports:
    - "127.0.0.1:8989:8989"
flaresolverr:
  # no ports published at all — only Prowlarr talks to it, by container name
```

```
https://docker-vm.<TAILNET_NAME>.ts.net:11443  -> qBittorrent
https://docker-vm.<TAILNET_NAME>.ts.net:12443  -> Radarr
https://docker-vm.<TAILNET_NAME>.ts.net:13443  -> Sonarr
https://docker-vm.<TAILNET_NAME>.ts.net:14443  -> Prowlarr
```

The 6881 torrent peer port is the one deliberate exception — it has to be
reachable by other peers on the internet to get decent download speeds, same
tradeoff as Minecraft's 25565 (see [Firewall](firewall.md)). Everything
*administrative* about this stack is Tailscale-only; only the actual P2P
traffic is exposed, and only on that one port.

## Prowlarr -> FlareSolverr: assign via Tags, not a "Proxy" field

Prowlarr's UI has no dropdown that says "Proxy" on an indexer's edit page —
easy to go looking for one and assume it's hidden behind an "Advanced"
toggle. Confirmed by inspecting Prowlarr's own SQLite schema directly
(`IndexerProxies` and `Indexers` tables) instead of guessing further: both
tables have a `Tags` column and nothing else links them. The actual flow:

1. **Settings -> Indexers -> Proxies** -> add FlareSolverr, give it a tag
   (e.g. `flaresolverr`)
2. Open the Cloudflare-protected indexer's settings -> add that same tag
3. Prowlarr routes that indexer's requests through FlareSolverr because the
   tags match — there's no explicit "attach proxy to indexer" control beyond that

## Prowlarr -> Radarr/Sonarr sync can silently no-op

Adding an indexer in Prowlarr and connecting an "Application" (Radarr/Sonarr)
are two separate steps — Prowlarr push the indexer list *to* each connected
app. This sync can fail silently:

- Radarr showed **0 indexers** even though Prowlarr's UI claimed everything
  was "fully synced."
- Root cause: a transient `429` from Prowlarr's own Torznab API during
  Radarr's own validation callback *into* Prowlarr, which made Radarr reject
  the indexer push with a `400`. Nothing retried automatically.
- Fix: once the rate limit cleared, force a real sync —
  `POST /api/v1/command` with `{"name":"ApplicationIndexerSync","forceSync":true}`.
  **The un-forced/default form is a silent no-op** if Prowlarr's internal
  state thinks nothing changed, even when the downstream app is actually
  out of sync. `forceSync: true` is the only reliable way to push a real
  resync from the UI/API.

**Takeaway:** if a `docker cp`'d SQLite file and the live app disagree,
trust the live app — SQLite's WAL (write-ahead log) means recent writes
often aren't in the base `.db` file yet. Every check in this stack that
looked wrong on a raw file copy turned out right when queried through the
app's actual API instead.

## Manual review is on, not full auto-grab

Radarr/Sonarr default to auto-grabbing the first release that matches your
quality profile the moment RSS or a scheduled search finds one — no human
in the loop. That default is off in this setup. Per-indexer, in Radarr:

```
PUT /api/v3/indexer/{id}
{
  "enableRss": false,
  "enableAutomaticSearch": false,
  "enableInteractiveSearch": true
}
```

`enableInteractiveSearch` stays **true** — you can still search and grab
manually from the UI, nothing about finding/downloading movies changed.
What's off is *unattended* grabbing. See "The auto-grab incident" below for
why this matters more than it sounds like it would.

## The auto-grab incident

With auto-grab still on, Radarr pulled the first "match" for a movie
straight off a public tracker (1337x) with no review — a `.exe` disguised as
a movie release, actively seeding to other peers by the time it was caught.

- **Found via:** a stuck download that "wouldn't refresh" turned into a
  full security audit (process list, network connections, SSH auth logs,
  every exposed port) rather than just poking at the download queue.
- **Removal was harder than expected.** Deleting the file from disk while
  qBittorrent was still running with it loaded in memory didn't work — its
  own periodic/shutdown "save resume data" routine rewrote the file and its
  `.fastresume`/`.torrent` state (in `BT_backup/`) right back, twice. The
  file only stayed gone after a full `docker stop` (halt the process
  entirely) **before** deleting both the media file and its resume-state
  files, then `docker start`.
- **No actual risk of execution:** confirmed the VM has no Wine and no
  relevant `binfmt_misc` handler — a Windows `.exe` had nowhere to run on a
  Linux-only Docker host regardless.
- **Fix:** manual review (above), so nothing lands in the download queue
  without a human looking at the release name/source first.

**Takeaway:** auto-grab-first-result on a public indexer is fine for
popular, well-seeded content but has no concept of "this filename looks
like malware" — that judgment has to come from a human reviewing the
release before it's grabbed, not after.

## qBittorrent WebUI: three compounding auth bugs

Getting qBittorrent's WebUI to actually log in, after moving it behind
Tailscale, took three separate fixes stacked on top of each other — each
one looked like "still broken" until isolated individually.

1. **The auto-generated temp password only works from true loopback.**
   qBittorrent prints a random temp password to its logs on first start
   with no password set. That password structurally only authenticates
   when the connecting source IP is literal `127.0.0.1`/`::1` **as seen by
   the qBittorrent process itself** — Docker's NAT/port-forwarding means
   even a request from the same host (let alone through Tailscale) never
   appears as true loopback to the containerized process. The only way to
   use it is `docker exec qbittorrent curl http://localhost:8080/...` —
   genuinely inside the container's network namespace, not `ssh` to the
   host, not a browser, not `curl` from outside the container.

2. **`WebUI\HostHeaderValidation` breaks any reverse-proxied/port-remapped
   access, independent of credentials.** This is a DNS-rebinding
   protection that rejects any request whose `Host` header port doesn't
   match qBittorrent's own internal listening port — which `tailscale
   serve`'s port remap (11443 externally -> 8080 internally) always
   trips. Confirmed via qBittorrent's *own* internal log
   (`/api/v2/log/main?last_known_id=-1`), which had the exact reason:
   `"WebUI: Invalid Host header, port mismatch. Server port: '8080'.
   Received Host header: 'docker-vm.<TAILNET_NAME>.ts.net:11443'"`.
   Fixed by setting `WebUI\HostHeaderValidation=false` directly in
   `qBittorrent.conf` (container stopped first, edited via a throwaway
   container mounting the same volume, then restarted).

3. **The brute-force IP ban is real, in-memory only, and not visible in
   any config file or preference.** Repeated failed logins (mine during
   testing, and separately the user's own browser trying the
   loopback-only temp password from outside the container) triggered:
   `"Your IP address has been banned after too many failed authentication
   attempts."` It's **not** in `qBittorrent.conf`'s `banned_IPs`
   preference or anywhere else on disk that could be found — it lives
   entirely in the running process's memory, keyed to whatever source IP
   qBittorrent thinks the request came from. Two consequences worth
   knowing:
   - **A container restart clears it** — the fast, reliable fix, since
     there's nothing persisted to undo.
   - **True-localhost access (`docker exec ... curl localhost:8080`) is
     unaffected by a ban on the external/proxied path** — confirmed by
     testing it directly while the WebUI ban was still active in a
     browser. Because all Tailscale-proxied traffic looks like it's
     coming from the same address to qBittorrent, a couple of mistyped
     login attempts is enough to re-trigger the ban for *everyone* using
     that path — but `docker exec` from the host always still works,
     since it's genuinely a different source as far as qBittorrent's
     process is concerned.

**Takeaway:** when a self-hosted app's WebUI fails in ways that don't match
"wrong password," check its own internal log before escalating — the
generic "Unauthorized" a browser shows can be masking a completely
different, more specific reason (host-header validation, a ban, a
loopback-only credential) that's only visible from inside the app itself.

## Setting a password without ever typing it to an AI assistant

Same principle as [Pi-hole's password handling](pihole.md): a real password
should never be chosen by, or ideally ever pass through, an AI assistant's
context. In practice for this stack:

- A **random bootstrap password** (never a real/reused one) was generated
  to get past the initial temp-password/localhost restriction, set via the
  qBittorrent API from inside the container.
- The **real, final password** is meant to be typed by the human directly —
  either into the WebUI login form once bootstrapped, or via a one-line
  `docker exec` command run in their own terminal, substituting their own
  value at the point of running it rather than asking an assistant to
  generate or apply it.
- Caught one real slip during this process: a password was pasted into
  chat that turned out to be a modified version of an **already-flagged,
  previously-compromised password** from the Pi-hole incident (same root,
  different punctuation). Reused/lightly-modified passwords defeat the
  point of rotating after a leak — a credential flagged as compromised
  should be replaced with something unrelated, not a variation on the same
  theme.

## Homarr integration

Same cross-network lesson as documented in [Homarr](homarr.md#lessons-learned):
Homarr and this stack are separate Compose projects on separate Docker
networks by default, so Homarr can't reach any of these containers by name
until it's explicitly joined to `media-stack_default` too:

```yaml
# homarr's compose file
networks:
  - default
  - uptime-kuma_default
  - pihole_default
  - media-stack_default   # added to reach radarr/sonarr/prowlarr/qbittorrent

networks:
  media-stack_default:
    external: true
```

Widget URLs in Homarr use the container **name** and **internal** port
(`http://radarr:7878`, not the Tailscale URL or the loopback-published
host port) — Homarr reaches these over the internal Docker network, not
through Tailscale.

## Lessons learned

- Prowlarr assigns a proxy (FlareSolverr) to an indexer via matching
  **Tags** on both sides — there's no direct "attach proxy" dropdown.
- Prowlarr's app sync (`ApplicationIndexerSync`) can silently no-op; use
  `forceSync: true` when you know something's actually out of sync.
- Never trust a `docker cp`'d SQLite file over the live app's own API —
  WAL-mode writes lag behind what's on disk.
- Auto-grab with no manual review is a real security exposure on public
  trackers, not just a quality-control nuisance — enable manual review
  (`enableRss`/`enableAutomaticSearch: false`, `enableInteractiveSearch:
  true`) by default.
- Deleting an in-use torrent's file doesn't stick until the process itself
  is stopped first — qBittorrent's own resume-data save will resurrect it.
- A self-hosted app's own internal log is worth checking before assuming
  "Unauthorized" means "wrong password" — host-header validation and
  IP bans both produce the same generic browser-facing error.
- IP bans that live only in a running process's memory clear on restart,
  and don't necessarily block every access path (true loopback vs. a
  reverse-proxied path can be treated as different source IPs).
- A password already flagged as compromised shouldn't be rotated to a
  lightly-modified version of itself — treat it as fully burned.
