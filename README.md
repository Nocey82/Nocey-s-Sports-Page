# Nocey-s-Sports-Page
# Create a small PATCH zip that adds:
# - LaLiga + UFC league support (frontend & backend)
# - Sample CNAME file (for custom domain)
# - Patch notes with copy/paste commands
#
# We include both `frontend/app.js` and a root-level `app.js` so the user can
# copy whichever matches their repo layout.

import os, zipfile, textwrap, json, pathlib

root = "/mnt/data/patch-laliga-ufc-cname"
os.makedirs(root, exist_ok=True)

patch_notes = """# Live Sports Hub — Patch: LaLiga + UFC + CNAME

This patch adds:
- **LaLiga** (Spain) across Scores, Standings, Rosters, Schedules.
- **UFC** (event-based) for Scores and Summary/Box (no team rosters/schedules).
- A **CNAME.sample** for GitHub Pages custom domain.

## How to apply

1) **Backend (Cloudflare Worker)**
   - Replace your `backend/worker.js` with the one in this patch.
   - Deploy:
     ```bash
     cd backend
     wrangler deploy
     ```

2) **Frontend**
   - If your site lives at the repo **root** (GitHub Pages root): copy `app.js` to repo root and commit.
   - If your site lives in **/frontend**: copy `frontend/app.js` there and commit.
   - Edit the top line to set your Worker URL:
     ```js
     const API_BASE = "https://scores-proxy.YOURNAME.workers.dev";
     ```
   - Commit & push:
     ```bash
     git add app.js frontend/app.js
     git commit -m "Add LaLiga + UFC + CNAME patch"
     git push
     ```

3) **Custom Domain (optional)**
   - Rename `CNAME.sample` → `CNAME`, edit it to a single line with your domain (e.g., `sports.example.com`), commit & push.
   - In your DNS, create a CNAME record pointing that host to `YOURNAME.github.io`.

## Verify
- Teams: `https://YOUR-WORKER/api/teams?league=laliga`
- Scoreboard today: `.../api/scoreboard?league=laliga&date=YYYY-MM-DD`
- UFC cards: `.../api/scoreboard?league=ufc&date=YYYY-MM-DD`
"""

cname_sample = "sports.example.com\n"

# Frontend app.js (top portion includes LEAGUES with LaLiga+UFC)
frontend_app_js = """// === CONFIG ===
const API_BASE = "https://scores-proxy.YOURNAME.workers.dev"; // ← set this to your Worker URL

// Add LaLiga + UFC to leagues list
const LEAGUES = {
  nfl: { sport: 'football', key: 'nfl', label: 'NFL' },
  nba: { sport: 'basketball', key: 'nba', label: 'NBA' },
  mlb: { sport: 'baseball', key: 'mlb', label: 'MLB' },
  nhl: { sport: 'hockey', key: 'nhl', label: 'NHL' },
  ncf: { sport: 'football', key: 'college-football', label: 'NCAA Football' },
  ncb: { sport: 'basketball', key: 'mens-college-basketball', label: "NCAA Men's Basketball" },
  wcb: { sport: 'basketball', key: 'womens-college-basketball', label: "NCAA Women's Basketball" },
  mls: { sport: 'soccer', key: 'usa.1', label: 'MLS' },
  epl: { sport: 'soccer', key: 'eng.1', label: 'Premier League' },
  laliga: { sport: 'soccer', key: 'esp.1', label: 'LaLiga' },   // NEW
  ufc: { sport: 'mma', key: 'ufc', label: 'UFC' },               // NEW (event-based)
};

// NOTE: The rest of your existing app.js can remain unchanged.
// As long as the UI builds the dropdown from LEAGUES and uses the existing
// endpoints (/api/scoreboard, /api/teams, etc.), LaLiga + UFC will appear.
"""

# Root-level app.js (same content for convenience)
root_app_js = frontend_app_js

# Backend Worker with leagueMapping including LaLiga + UFC; routes kept broad
worker_js = """// Cloudflare Worker — ESPN proxy with LaLiga + UFC
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const path = url.pathname;

    if (request.method === 'OPTIONS') return new Response(null, { headers: cors() });

    try {
      if (path === '/api/scoreboard') {
        const league = url.searchParams.get('league') || 'nfl';
        const date = url.searchParams.get('date'); // YYYY-MM-DD
        const map = leagueMapping(league);
        if (!map) return json({ error: 'Unsupported league' }, 400);
        const yyyymmdd = date ? date.replaceAll('-', '') : currentDateYYYYMMDD();
        const endpoint = `https://site.api.espn.com/apis/site/v2/sports/${map.sport}/${map.key}/scoreboard?dates=${yyyymmdd}`;
        return await pass(endpoint, 30);
      }

      if (path === '/api/teams') {
        const league = url.searchParams.get('league') || 'nfl';
        const map = leagueMapping(league);
        if (!map) return json({ error: 'Unsupported league' }, 400);
        // UFC is event-based, no team list
        if (league === 'ufc') return json({ sports: [] }, 200);
        const endpoint = `https://site.api.espn.com/apis/site/v2/sports/${map.sport}/${map.key}/teams`;
        return await pass(endpoint, 3600);
      }

      if (path === '/api/roster') {
        const league = url.searchParams.get('league') || 'nfl';
        const teamId = url.searchParams.get('teamId');
        if (!teamId) return json({ error: 'Missing teamId' }, 400);
        const map = leagueMapping(league);
        if (!map) return json({ error: 'Unsupported league' }, 400);
        const endpoint = `https://site.api.espn.com/apis/site/v2/sports/${map.sport}/${map.key}/teams/${teamId}?enable=roster`;
        return await pass(endpoint, 600);
      }

      if (path === '/api/standings') {
        const league = url.searchParams.get('league') || 'nfl';
        const map = leagueMapping(league);
        if (!map) return json({ error: 'Unsupported league' }, 400);
        const endpoint = `https://site.api.espn.com/apis/site/v2/sports/${map.sport}/${map.key}/standings`;
        const resp = await fetch(endpoint, { headers: commonHeaders(), cf: { cacheEverything: true, cacheTtl: 300 }});
        if (resp.ok) {
          const data = await resp.json();
          return json(data, 200, { 'Cache-Control': 'public, max-age=60, s-maxage=300' });
        } else {
          const alt = `https://site.web.api.espn.com/apis/v2/sports/${map.sport}/${map.key}/standings`;
          return await pass(alt, 300);
        }
      }

      if (path === '/api/summary') {
        const league = url.searchParams.get('league') || 'nfl';
        const eventId = url.searchParams.get('eventId');
        if (!eventId) return json({ error: 'Missing eventId' }, 400);
        const map = leagueMapping(league);
        if (!map) return json({ error: 'Unsupported league' }, 400);
        const endpoint = `https://site.web.api.espn.com/apis/v2/sports/${map.sport}/${map.key}/summary?event=${eventId}`;
        return await pass(endpoint, 30);
      }

      if (path === '/api/boxscore') {
        const league = url.searchParams.get('league') || 'nfl';
        const eventId = url.searchParams.get('eventId');
        if (!eventId) return json({ error: 'Missing eventId' }, 400);
        const map = leagueMapping(league);
        if (!map) return json({ error: 'Unsupported league' }, 400);
        const endpoint = `https://site.web.api.espn.com/apis/v2/sports/${map.sport}/${map.key}/boxscore?event=${eventId}`;
        return await pass(endpoint, 30);
      }

      if (path === '/api/athlete') {
        const league = url.searchParams.get('league') || 'nfl';
        const athleteId = url.searchParams.get('athleteId');
        if (!athleteId) return json({ error: 'Missing athleteId' }, 400);
        const map = leagueMapping(league);
        if (!map) return json({ error: 'Unsupported league' }, 400);
        const endpoint = `https://site.web.api.espn.com/apis/common/v3/sports/${map.sport}/${map.key}/athletes/${athleteId}/statistics`;
        return await pass(endpoint, 300);
      }

      if (path === '/api/schedule') {
        const league = url.searchParams.get('league') || 'nfl';
        const teamId = url.searchParams.get('teamId');
        const season = url.searchParams.get('season');
        if (!teamId) return json({ error: 'Missing teamId' }, 400);
        // UFC has no team schedule
        if (league === 'ufc') return json({ events: [] }, 200);
        const map = leagueMapping(league);
        if (!map) return json({ error: 'Unsupported league' }, 400);
        const endpoint = `https://site.api.espn.com/apis/site/v2/sports/${map.sport}/${map.key}/teams/${teamId}/schedule${season?`?season=${season}`:''}`;
        return await pass(endpoint, 300);
      }

      if (path === '/api/health') {
        return json({ ok: true, time: new Date().toISOString() });
      }

      return new Response('Not found', { status: 404, headers: cors() });
    } catch (e) {
      return json({ error: 'Unhandled error', message: String(e) }, 500);
    }
  }
};

function leagueMapping(id) {
  const map = {
    nfl: { sport: 'football', key: 'nfl' },
    nba: { sport: 'basketball', key: 'nba' },
    mlb: { sport: 'baseball', key: 'mlb' },
    nhl: { sport: 'hockey', key: 'nhl' },
    ncf: { sport: 'football', key: 'college-football' },
    ncb: { sport: 'basketball', key: 'mens-college-basketball' },
    wcb: { sport: 'basketball', key: 'womens-college-basketball' },
    mls: { sport: 'soccer', key: 'usa.1' },
    epl: { sport: 'soccer', key: 'eng.1' },
    laliga: { sport: 'soccer', key: 'esp.1' }, // NEW
    ufc: { sport: 'mma', key: 'ufc' },        // NEW (event-based)
  };
  return map[id];
}

function currentDateYYYYMMDD(){
  const now = new Date();
  const yyyy = now.getFullYear();
  const mm = String(now.getMonth()+1).padStart(2,'0');
  const dd = String(now.getDate()).padStart(2,'0');
  return `${yyyy}${mm}${dd}`;
}

function commonHeaders(){
  return {
    'User-Agent': 'Mozilla/5.0 (compatible; ScoreProxy/1.3)',
    'Accept': 'application/json, text/plain, */*',
  };
}

async function pass(endpoint, sMax=60){
  const resp = await fetch(endpoint, { headers: commonHeaders(), cf: { cacheEverything: true, cacheTtl: sMax }});
  if (!resp.ok) return json({ error: 'Upstream error', status: resp.status }, resp.status);
  const data = await resp.json();
  return json(data, 200, { 'Cache-Control': `public, max-age=${Math.min(30,sMax)}, s-maxage=${sMax}` });
}

function cors(extra={}){
  return {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    ...extra
  };
}

function json(obj, status=200, extra={}){
  return new Response(JSON.stringify(obj), {
    status,
    headers: { 'Content-Type': 'application/json; charset=utf-8', ...cors(extra) }
  });
}
"""

# Write files into the patch folder
with open(os.path.join(root, "PATCH_NOTES.md"), "w", encoding="utf-8") as f:
    f.write(patch_notes)
with open(os.path.join(root, "CNAME.sample"), "w", encoding="utf-8") as f:
    f.write(cname_sample)

# Both app.js variants
os.makedirs(os.path.join(root, "frontend"), exist_ok=True)
with open(os.path.join(root, "frontend", "app.js"), "w", encoding="utf-8") as f:
    f.write(frontend_app_js)
with open(os.path.join(root, "app.js"), "w", encoding="utf-8") as f:
    f.write(root_app_js)

# Backend worker
os.makedirs(os.path.join(root, "backend"), exist_ok=True)
with open(os.path.join(root, "backend", "worker.js"), "w", encoding="utf-8") as f:
    f.write(worker_js)

# Zip it
zip_path = "/mnt/data/patch-laliga-ufc-cname.zip"
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
    for folder, _, files in os.walk(root):
        for file in files:
            file_path = os.path.join(folder, file)
            arcname = os.path.relpath(file_path, root)
            zipf.write(file_path, arcname)

zip_path