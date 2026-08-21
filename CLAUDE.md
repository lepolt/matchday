# Matchday

A single-file HTML/CSS/JS dashboard that tracks multiple youth soccer teams'
schedules from GotSport, so the owner can see all their kids' upcoming games,
recent results, and records in one place without visiting each team's page.

- **Repo:** https://github.com/lepolt/matchday
- **Live site:** GitHub Pages, served from `main` branch root as `index.html`
  (https://lepolt.github.io/matchday/)
- **File:** everything lives in one file, `index.html` — no build step, no
  dependencies, no framework. Plain HTML + CSS + vanilla JS.

## Why it's built this way

GotSport (`system.gotsport.com`) disallows automated scraping via
`robots.txt` and its footer states "copying of content from this site
without permission is a violation of our Terms of Use." This tool only
reads publicly-visible schedule data for the owner's own kids' teams,
on-demand (not continuous crawling), for personal tracking — not
republishing or bulk harvesting. Worth keeping in mind if this project ever
grows beyond personal use.

Because of that, and because GotSport's server doesn't send CORS headers
allowing cross-origin browser fetches, the page can't fetch GotSport
directly. It routes fetches through public CORS relay services instead.
This only works when the page is served over a real origin (`http://` or
`https://`) — it does **not** work if the file is opened directly via
`file://`, because that sends `Origin: null`, which the relays reject.

## Data flow

1. User adds a team via "Manage teams": a label (e.g. "Maya - U14 Girls")
   plus the team's GotSport schedule URL
   (`system.gotsport.com/org_event/events/<event>/schedules?team=<id>`).
2. On page load (and on "Refresh all"), each team's URL is fetched through a
   chain of CORS proxies, tried in order until one returns something that
   looks like a real GotSport page (checked via a regex for
   `<html|widget-body|GotSport`, to reject proxy error payloads):
   1. `api.allorigins.win/raw?url=`
   2. `api.codetabs.com/v1/proxy?quest=`
   3. `corsproxy.io/?url=` — **only works from `localhost` origins on its
      free tier**, so it's ordered last since the live site is on
      `github.io`. If this project is ever tested locally again via
      `python3 -m http.server`, this one will actually succeed first in
      practice even though it's listed third (the earlier ones are tried
      first and may simply fail faster there).
   4. `api.allorigins.win` again, cache-busted, as a last resort.
3. The returned HTML is parsed client-side with `DOMParser`. Match tables
   are identified by a `<th>` containing "Match #" (this is how GotSport
   marks its schedule tables; the standings table uses similar CSS classes
   but different headers, so header text is the reliable discriminator).
4. Each match row yields: match #, date/time, home team, result, away team,
   location, division. The team's own name (scraped from the page's
   `h4.no-margin-top`) is used to determine which side is "us" vs the
   opponent, and win/loss/draw by parsing the score out of the result cell.

## Storage

The team list itself lives in the **URL fragment** (`#teams=<url-encoded
JSON array of {id, name, url}>`), not `localStorage`. `saveTeams()` writes
it via `history.replaceState` on every add/rename/reorder/remove;
`loadTeams()` reads it back out of `location.hash` on load. A fragment is
never sent to the server, so it isn't subject to GitHub Pages/Fastly
request-line length limits the way a `?query=` param would be — the only
real ceiling is the browser/bookmark manager's own, far beyond what a
personal team list needs.

This makes the URL itself a fully portable, bookmarkable dashboard: bookmark
the current link (e.g. via Safari, synced by iCloud) on another device and
it reproduces the exact same teams/names/order, no re-adding required. It
also means multiple independent "dashboards" are just multiple different
bookmarked links — e.g. one with every kid's team, another with just one
team to send to a grandparent. A "Copy link" button in the controls row
copies `location.href` to the clipboard as a convenience, since the address
bar already always reflects the live, current dashboard.

**Editing is per-link, not synced.** Since there's no backend, adding or
removing a team only updates the fragment of the tab you're editing in —
it does not retroactively update any other bookmark you'd already saved
elsewhere. After editing your team list, re-save/re-share whichever
bookmark(s) you want to reflect the change.

`localStorage`, namespaced with a `matchday:` prefix, is still used for one
thing:

- `matchday:schedule:<id>` — last successfully parsed schedule for that
  team, shown immediately on load before the background refresh completes.
  Per-origin and purely a perf cache; unrelated to which dashboard/team-list
  is currently loaded.

**Migration:** pre-existing installs had the team list in a
`matchday:teams` localStorage key. `migrateLegacyTeams()` runs once on
load — if there's no `#teams=` fragment yet and that legacy key has data, it
converts it into a fragment (so the existing owner's team list survives as
a bookmarkable link) and removes the old key either way.

## Features implemented

- Add / rename / remove / reorder (▲▼) teams, all persisted via the URL
  fragment (see Storage above).
- "Copy link" button to grab the current dashboard's shareable/bookmarkable
  URL.
- Auto-refreshes all teams on page load; manual "Refresh all" button.
- Per-team card: W-L-D record, last match result, last-5-results as
  colored form circles (green/red/yellow, oldest → newest, hover for
  detail), next upcoming match highlighted, expandable full schedule,
  link back to the team's GotSport page.
- "Next 5 days" panel: upcoming games across all teams, grouped by
  calendar date (short format, e.g. "Sun, Aug 23"), team name shown before
  opponent, days with no games omitted.
- Per-team loading/error/ok status dot; errors show a retry-friendly
  message with a direct link to the GotSport page as fallback.

## Auth / deployment notes

- Pushed via SSH using a dedicated personal key (not the owner's work key —
  GitHub doesn't allow one public key on two accounts anyway). SSH config
  alias in `~/.ssh/config`:
  ```
  Host github-personal
      HostName github.com
      User git
      IdentityFile ~/.ssh/id_ed25519_personal
  ```
  Remote is set to `git@github-personal:lepolt/matchday.git`, not the
  default `github.com` host.

## Possible next steps (not yet requested/built)

Nothing currently pending — last request was moving the team list from
`localStorage` into the URL fragment for cross-device/no-backend syncing,
which is done. If picking this back up, good candidates might be: combining
the "Next 5 days" and per-team next-match views so they don't repeat info,
letting the CORS proxy list be user-editable in case all three relays
degrade, or replacing the public relays with a small self-hosted proxy
(e.g. a Cloudflare Worker) for reliability.
