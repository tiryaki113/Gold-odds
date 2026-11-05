// GOLD ODDS — SportScore public sports-data API proxy (Node.js 18+)
// SportScore requires a visible dofollow "Powered by SportScore" link on pages using its free data.
const http = require('node:http');
const fs = require('node:fs');
const path = require('node:path');
const PORT = Number(process.env.PORT || 3000);
const ROOT = __dirname;
const API = 'https://sportscore.com/api/widget/matches/';
const CACHE_MS = 60_000; // SportScore edge cache is also 60 seconds.
let cache = { at: 0, payload: null };
function send(res, code, data, type='application/json; charset=utf-8') {
  res.writeHead(code, { 'Content-Type': type, 'Cache-Control': 'no-store' });
  res.end(type.startsWith('application/json') ? JSON.stringify(data) : data);
}
function normalizeMatch(m, i) {
  const rawStatus = String(m.status || m.status_text || 'upcoming');
  const live = /live|in.?play|1st half|2nd half|half.?time|extra.?time|penalt/i.test(rawStatus);
  const finished = /finished|ended|full.?time|complete/i.test(rawStatus);
  const homeScore = m.home_score;
  const awayScore = m.away_score;
  const hasScore = homeScore !== null && homeScore !== undefined && awayScore !== null && awayScore !== undefined;
  return {
    id: m.id || m.match_id || m.url || `${m.home || 'home'}-${m.away || 'away'}-${m.time || i}`,
    home: m.home || 'Ev sahibi',
    away: m.away || 'Deplasman',
    league: m.competition || m.league || 'Futbol',
    starting_at: m.time || m.starting_at || '',
    status: live ? 'Canlı' : finished ? 'Tamamlandı' : (m.status_text || 'Başlamadı'),
    is_live: live,
    score: hasScore ? `${homeScore} - ${awayScore}` : '',
    home_logo: m.home_logo || '',
    away_logo: m.away_logo || '',
    match_url: m.url || ''
  };
}
async function getFixtures() {
  if (cache.payload && Date.now() - cache.at < CACHE_MS) return cache.payload;
  const url = new URL(API);
  url.searchParams.set('sport', 'football');
  url.searchParams.set('limit', '50');
  url.searchParams.set('src', 'gold-odds');
  const response = await fetch(url, { headers: { Accept: 'application/json' }, signal: AbortSignal.timeout(15000) });
  const body = await response.json().catch(() => ({}));
  if (!response.ok) throw new Error(body.error || `SportScore HTTP ${response.status}`);
  if (!Array.isArray(body.matches)) throw new Error('SportScore yanıtında matches listesi bulunamadı.');
  const fixtures = body.matches.map(normalizeMatch).sort((a,b) => {
    if (a.is_live !== b.is_live) return a.is_live ? -1 : 1;
    return String(a.starting_at).localeCompare(String(b.starting_at));
  });
  cache = { at: Date.now(), payload: { provider: 'SportScore', updated_at: body.updated || new Date().toISOString(), fixtures } };
  return cache.payload;
}
http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://${req.headers.host || 'localhost'}`);
  if (req.method === 'GET' && url.pathname === '/api/health') {
    return send(res, 200, { ok: true, provider: 'SportScore', configured: true, apiKeyRequired: false });
  }
  if (req.method === 'GET' && url.pathname === '/api/fixtures') {
    try { return send(res, 200, await getFixtures()); }
    catch (e) { return send(res, 502, { error: `SportScore verisi alınamadı: ${e.message}` }); }
  }
  if (req.method === 'GET' && (url.pathname === '/' || url.pathname === '/index.html')) {
    return send(res, 200, fs.readFileSync(path.join(ROOT, 'index.html')), 'text/html; charset=utf-8');
  }
  return send(res, 404, { error: 'Bulunamadı' });
}).listen(PORT, () => console.log(`GOLD ODDS / SportScore: http://localhost:${PORT}`));
