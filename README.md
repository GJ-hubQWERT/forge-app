import { useState, useCallback, useEffect, useRef } from "react";

// ─── DATE UTILS ───────────────────────────────────────────────────────────────
const TODAY = () => new Date().toISOString().split("T")[0];
const getDateStr = (offset = 0) => {
  const d = new Date(); d.setDate(d.getDate() + offset);
  return d.toISOString().split("T")[0];
};
const getWeekDates = () => Array.from({ length: 7 }, (_, i) => getDateStr(-(6 - i)));
const getDayOfWeek = (ds) => new Date(ds + "T12:00:00").toLocaleDateString("en-US", { weekday: "short" });
const formatTime = (s) => {
  const h = Math.floor(s / 3600), m = Math.floor((s % 3600) / 60), sec = s % 60;
  if (h > 0) return `${h}:${String(m).padStart(2, "0")}:${String(sec).padStart(2, "0")}`;
  return `${String(m).padStart(2, "0")}:${String(sec).padStart(2, "0")}`;
};
const formatDist = (m) => m >= 1000 ? `${(m / 1000).toFixed(2)} km` : `${m} m`;
const daysSince = (ds) => {
  if (!ds) return 999;
  return Math.floor((new Date() - new Date(ds + "T12:00:00")) / 86400000);
};

// ─── STORAGE ──────────────────────────────────────────────────────────────────
function useStorage(key, def) {
  const [val, setVal] = useState(() => {
    try { const s = localStorage.getItem(key); return s ? JSON.parse(s) : def; }
    catch { return def; }
  });
  const save = useCallback((v) => {
    const next = typeof v === "function" ? v(val) : v;
    setVal(next);
    try { localStorage.setItem(key, JSON.stringify(next)); } catch {}
  }, [key, val]);
  return [val, save];
}

// ─── STREAK CALC ──────────────────────────────────────────────────────────────
function calcStreak(dates) {
  // dates = array of "YYYY-MM-DD" strings when activity happened
  const set = new Set(dates);
  let streak = 0, d = new Date();
  // if today not done, start from yesterday
  if (!set.has(d.toISOString().split("T")[0])) d.setDate(d.getDate() - 1);
  while (true) {
    const ds = d.toISOString().split("T")[0];
    if (set.has(ds)) { streak++; d.setDate(d.getDate() - 1); }
    else break;
  }
  return streak;
}

function weeklyScore(activityDates, planned = 7) {
  const week = getWeekDates();
  const set = new Set(activityDates);
  const done = week.filter(d => set.has(d)).length;
  return Math.round((done / Math.min(planned, 7)) * 100);
}

// ─── STYLES ───────────────────────────────────────────────────────────────────
const STYLES = `
@import url('https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&family=Sora:wght@300;400;500;600;700&display=swap');

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --bg:       #0c0d0f;
  --s1:       #111317;
  --s2:       #171a1f;
  --s3:       #1d2128;
  --border:   #262b33;
  --border2:  #2f3540;
  --amber:    #f59e0b;
  --amber2:   #fbbf24;
  --amber-dim:#7c4f06;
  --text:     #e8eaed;
  --muted:    #6b7280;
  --muted2:   #9ca3af;
  --danger:   #ef4444;
  --green:    #10b981;
  --blue:     #3b82f6;
  --r:        12px;
  --r2:       20px;
}

html, body { height: 100%; background: var(--bg); }
body { color: var(--text); font-family: 'Sora', sans-serif; -webkit-font-smoothing: antialiased; overflow-x: hidden; }
.app { max-width: 430px; margin: 0 auto; min-height: 100vh; position: relative; background: var(--bg); }

/* ── TYPOGRAPHY ── */
.mono { font-family: 'Space Mono', monospace; }
.h1 { font-size: 42px; font-weight: 700; line-height: 1; letter-spacing: -1px; }
.h2 { font-size: 28px; font-weight: 700; line-height: 1.1; }
.h3 { font-size: 20px; font-weight: 600; }
.label { font-size: 10px; font-weight: 600; letter-spacing: 2px; text-transform: uppercase; color: var(--muted); }
.caption { font-size: 12px; color: var(--muted2); }

/* ── AUTH ── */
.auth-wrap { min-height: 100vh; display: flex; flex-direction: column; justify-content: flex-end; padding: 0; position: relative; overflow: hidden; }
.auth-bg { position: absolute; inset: 0; background: radial-gradient(ellipse at 20% 50%, rgba(245,158,11,0.06) 0%, transparent 60%), radial-gradient(ellipse at 80% 20%, rgba(245,158,11,0.04) 0%, transparent 50%); pointer-events: none; }
.auth-grid { position: absolute; inset: 0; background-image: linear-gradient(var(--border) 1px, transparent 1px), linear-gradient(90deg, var(--border) 1px, transparent 1px); background-size: 40px 40px; opacity: 0.3; pointer-events: none; }
.auth-content { position: relative; padding: 0 24px 48px; }
.auth-logo-wrap { padding: 60px 24px 40px; }
.forge-logo { font-family: 'Space Mono', monospace; font-size: 52px; font-weight: 700; color: var(--amber); letter-spacing: -2px; }
.forge-sub { font-size: 13px; color: var(--muted); letter-spacing: 1px; margin-top: 4px; }
.auth-card { background: var(--s1); border: 1px solid var(--border); border-radius: var(--r2); padding: 24px; }
.auth-title { font-size: 22px; font-weight: 700; margin-bottom: 20px; }
.field { margin-bottom: 14px; }
.field label { display: block; font-size: 11px; font-weight: 600; color: var(--muted2); letter-spacing: 1px; text-transform: uppercase; margin-bottom: 6px; }
.inp { width: 100%; background: var(--s2); border: 1px solid var(--border); color: var(--text); padding: 13px 14px; border-radius: var(--r); font-family: 'Sora', sans-serif; font-size: 15px; outline: none; transition: border-color 0.2s; }
.inp:focus { border-color: var(--amber); }
.inp::placeholder { color: var(--muted); }
.btn { width: 100%; background: var(--amber); color: #000; border: none; padding: 15px; border-radius: var(--r); font-family: 'Sora', sans-serif; font-size: 15px; font-weight: 700; cursor: pointer; letter-spacing: 0.3px; transition: opacity 0.15s, transform 0.1s; }
.btn:active { transform: scale(0.98); }
.btn:hover { opacity: 0.92; }
.btn-ghost { background: transparent; color: var(--muted2); border: 1px solid var(--border); font-weight: 500; }
.btn-sm { padding: 9px 16px; font-size: 13px; font-weight: 600; width: auto; border-radius: 10px; }
.btn-outline { background: transparent; border: 1px solid var(--amber); color: var(--amber); font-weight: 600; }
.btn-danger { background: var(--danger); color: #fff; }
.btn-green { background: var(--green); color: #fff; }
.auth-toggle { text-align: center; font-size: 13px; color: var(--muted); margin-top: 14px; }
.auth-toggle span { color: var(--amber); cursor: pointer; font-weight: 600; }
.err-msg { color: var(--danger); font-size: 13px; margin-bottom: 10px; padding: 10px 14px; background: rgba(239,68,68,0.08); border-radius: 8px; border: 1px solid rgba(239,68,68,0.2); }

/* ── NAV ── */
.nav { position: fixed; bottom: 0; left: 50%; transform: translateX(-50%); width: 100%; max-width: 430px; background: var(--s1); border-top: 1px solid var(--border); display: flex; z-index: 100; padding: 0 0 env(safe-area-inset-bottom); }
.nav-item { flex: 1; display: flex; flex-direction: column; align-items: center; padding: 10px 0 13px; gap: 3px; cursor: pointer; color: var(--muted); transition: color 0.15s; position: relative; }
.nav-item.active { color: var(--amber); }
.nav-label { font-size: 9px; font-weight: 600; letter-spacing: 0.8px; text-transform: uppercase; }
.nav-pip { width: 3px; height: 3px; border-radius: 50%; background: var(--amber); position: absolute; bottom: 6px; opacity: 0; transition: opacity 0.2s; }
.nav-item.active .nav-pip { opacity: 1; }

/* ── SCREEN ── */
.screen { padding: 20px 16px 100px; }
.screen-hd { margin-bottom: 28px; }
.screen-title { font-size: 32px; font-weight: 700; letter-spacing: -0.5px; line-height: 1; }
.screen-sub { font-size: 13px; color: var(--muted2); margin-top: 5px; }

/* ── CARDS ── */
.card { background: var(--s1); border: 1px solid var(--border); border-radius: var(--r); padding: 18px; margin-bottom: 12px; }
.card-hd { display: flex; justify-content: space-between; align-items: center; margin-bottom: 14px; }
.card-hd-title { font-size: 11px; font-weight: 600; letter-spacing: 1.5px; text-transform: uppercase; color: var(--muted); }

/* ── PBAR ── */
.pbar { background: var(--s3); border-radius: 3px; height: 5px; overflow: hidden; }
.pbar-fill { height: 100%; border-radius: 3px; background: var(--amber); transition: width 0.5s ease; }
.pbar-fill.green { background: var(--green); }
.pbar-fill.blue { background: var(--blue); }
.pbar-fill.danger { background: var(--danger); }

/* ── STAT BOXES ── */
.stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 12px; }
.stat-box { background: var(--s1); border: 1px solid var(--border); border-radius: var(--r); padding: 16px; }
.stat-num { font-family: 'Space Mono', monospace; font-size: 40px; font-weight: 700; color: var(--amber); line-height: 1; }
.stat-lbl { font-size: 10px; font-weight: 600; letter-spacing: 1.5px; text-transform: uppercase; color: var(--muted); margin-top: 5px; }
.stat-sub { font-size: 12px; color: var(--muted2); margin-top: 3px; }

/* ── WEEK BAR CHART ── */
.week-bars { display: flex; gap: 5px; align-items: flex-end; height: 60px; }
.wb-col { flex: 1; display: flex; flex-direction: column; align-items: center; gap: 4px; height: 100%; justify-content: flex-end; }
.wb-bar { width: 100%; border-radius: 3px 3px 0 0; min-height: 3px; background: var(--s3); transition: height 0.4s ease; }
.wb-bar.on { background: var(--amber); }
.wb-bar.today { background: var(--amber2); }
.wb-day { font-size: 9px; font-weight: 600; color: var(--muted); letter-spacing: 0.5px; }

/* ── WEEK DOTS ── */
.wdots { display: flex; gap: 4px; }
.wdot { flex: 1; height: 4px; border-radius: 2px; background: var(--s3); }
.wdot.on { background: var(--amber); }
.wdot.today { background: var(--amber2); }

/* ── BIG ACTION BUTTON ── */
.big-btn-wrap { display: flex; justify-content: center; margin: 28px 0; }
.big-btn { width: 140px; height: 140px; border-radius: 50%; background: var(--amber); color: #000; border: none; cursor: pointer; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 4px; font-family: 'Sora', sans-serif; font-weight: 700; font-size: 14px; letter-spacing: 0.5px; box-shadow: 0 0 40px rgba(245,158,11,0.2), 0 8px 32px rgba(0,0,0,0.4); transition: transform 0.15s, box-shadow 0.15s; }
.big-btn:active { transform: scale(0.95); }
.big-btn:hover { box-shadow: 0 0 60px rgba(245,158,11,0.3), 0 8px 32px rgba(0,0,0,0.4); }
.big-btn.active { background: var(--s2); color: var(--danger); border: 2px solid var(--danger); box-shadow: 0 0 40px rgba(239,68,68,0.15); }
.big-btn.active:hover { box-shadow: 0 0 60px rgba(239,68,68,0.25); }
.big-btn-icon { font-size: 32px; line-height: 1; }

/* ── LIVE DISPLAY ── */
.live-display { text-align: center; padding: 20px 0; }
.live-time { font-family: 'Space Mono', monospace; font-size: 56px; font-weight: 700; color: var(--text); letter-spacing: -2px; line-height: 1; }
.live-sub { font-family: 'Space Mono', monospace; font-size: 24px; color: var(--amber); margin-top: 8px; }
.live-label { font-size: 11px; font-weight: 600; letter-spacing: 2px; text-transform: uppercase; color: var(--muted); margin-top: 4px; }

/* ── MODAL / SHEET ── */
.overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.85); z-index: 200; display: flex; align-items: flex-end; justify-content: center; backdrop-filter: blur(2px); }
.sheet { background: var(--s1); border-top: 1px solid var(--border); border-radius: 22px 22px 0 0; padding: 28px 20px 44px; width: 100%; max-width: 430px; max-height: 90vh; overflow-y: auto; }
.sheet-title { font-size: 24px; font-weight: 700; letter-spacing: -0.3px; margin-bottom: 22px; }
.sheet-handle { width: 36px; height: 4px; background: var(--border2); border-radius: 2px; margin: 0 auto 20px; }

/* ── TAGS ── */
.tags { display: flex; gap: 6px; flex-wrap: wrap; }
.tag { background: var(--s2); border: 1px solid var(--border); color: var(--muted2); padding: 8px 14px; border-radius: 8px; font-size: 13px; font-weight: 500; cursor: pointer; transition: all 0.15s; }
.tag.on { border-color: var(--amber); color: var(--amber); background: rgba(245,158,11,0.08); }

/* ── LIST ITEMS ── */
.list-item { background: var(--s1); border: 1px solid var(--border); border-radius: var(--r); padding: 14px 16px; margin-bottom: 8px; display: flex; justify-content: space-between; align-items: center; }
.list-item-left { flex: 1; }
.list-item-name { font-size: 15px; font-weight: 600; }
.list-item-meta { font-size: 12px; color: var(--muted2); margin-top: 3px; }
.list-item-right { display: flex; flex-direction: column; align-items: flex-end; gap: 4px; }

/* ── BADGES ── */
.badge { padding: 3px 10px; border-radius: 20px; font-size: 10px; font-weight: 700; letter-spacing: 1px; text-transform: uppercase; display: inline-block; }
.badge-amber { background: rgba(245,158,11,0.1); color: var(--amber); border: 1px solid rgba(245,158,11,0.2); }
.badge-green { background: rgba(16,185,129,0.1); color: var(--green); border: 1px solid rgba(16,185,129,0.2); }
.badge-blue { background: rgba(59,130,246,0.1); color: var(--blue); border: 1px solid rgba(59,130,246,0.2); }
.badge-red { background: rgba(239,68,68,0.1); color: var(--danger); border: 1px solid rgba(239,68,68,0.2); }

/* ── TOAST ── */
.toast { position: fixed; top: 20px; left: 50%; transform: translateX(-50%); background: var(--s1); border: 1px solid var(--amber); color: var(--amber); font-size: 13px; font-weight: 600; padding: 10px 20px; border-radius: 20px; z-index: 400; animation: toastin 0.25s ease; white-space: nowrap; box-shadow: 0 4px 20px rgba(0,0,0,0.4); }
@keyframes toastin { from { opacity: 0; transform: translate(-50%, -8px); } to { opacity: 1; transform: translate(-50%, 0); } }

/* ── MISC ── */
.divider { height: 1px; background: var(--border); margin: 16px 0; }
.empty { text-align: center; padding: 44px 20px; color: var(--muted); font-size: 14px; }
.empty-icon { font-size: 40px; margin-bottom: 10px; }
.row { display: flex; gap: 10px; }
.col { flex: 1; }
.fab { position: fixed; bottom: 82px; right: calc(50% - 215px + 16px); width: 52px; height: 52px; border-radius: 50%; background: var(--amber); color: #000; border: none; cursor: pointer; display: flex; align-items: center; justify-content: center; font-size: 22px; box-shadow: 0 4px 20px rgba(245,158,11,0.3); z-index: 50; transition: transform 0.15s; }
.fab:active { transform: scale(0.9); }
@media (max-width: 430px) { .fab { right: 16px; } }
.section-hd { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
.streak-badge { display: inline-flex; align-items: center; gap: 5px; background: rgba(245,158,11,0.08); border: 1px solid rgba(245,158,11,0.2); border-radius: 20px; padding: 5px 12px; font-size: 13px; font-weight: 600; color: var(--amber); }
.summary-card { background: linear-gradient(135deg, rgba(245,158,11,0.06) 0%, rgba(245,158,11,0.02) 100%); border: 1px solid rgba(245,158,11,0.2); border-radius: var(--r); padding: 20px; margin-bottom: 12px; }
.checklist-item { display: flex; align-items: center; gap: 12px; padding: 12px 0; border-bottom: 1px solid var(--border); }
.checklist-item:last-child { border-bottom: none; }
.check-circle { width: 28px; height: 28px; border-radius: 50%; border: 2px solid var(--border2); display: flex; align-items: center; justify-content: center; flex-shrink: 0; transition: all 0.2s; }
.check-circle.done { background: var(--amber); border-color: var(--amber); color: #000; }
.check-icon { font-size: 13px; }
.checklist-label { font-size: 14px; font-weight: 500; flex: 1; }
.checklist-value { font-size: 12px; font-weight: 600; color: var(--muted2); font-family: 'Space Mono', monospace; }
select.inp { appearance: none; }
textarea.inp { resize: none; min-height: 80px; }
.focus-ring { border-radius: 50%; width: 180px; height: 180px; display: flex; align-items: center; justify-content: center; flex-direction: column; margin: 0 auto; position: relative; }
.focus-ring svg { position: absolute; inset: 0; transform: rotate(-90deg); }
.focus-center { position: relative; z-index: 1; text-align: center; }
.nudge-banner { background: rgba(239,68,68,0.08); border: 1px solid rgba(239,68,68,0.2); border-radius: var(--r); padding: 12px 16px; margin-bottom: 12px; font-size: 13px; color: var(--danger); }
`;

// ─── SVG ICONS ────────────────────────────────────────────────────────────────
const Ic = ({ d, size = 22, stroke = "currentColor" }) => (
  <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={stroke} strokeWidth="1.8" strokeLinecap="round" strokeLinejoin="round">
    {Array.isArray(d) ? d.map((p, i) => <path key={i} d={p} />) : <path d={d} />}
  </svg>
);
const Icons = {
  home:    "M3 9l9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z M9 22V12h6v10",
  run:     ["M13 4a1 1 0 1 0 2 0 1 1 0 0 0-2 0", "M7.5 14.5s1.5-1 3-1 3.5 2 5 2 3-1 3-1", "M5 19l2.5-4.5L10 17l2-4 2.5 2.5L17 9"],
  book:    "M4 19.5A2.5 2.5 0 0 1 6.5 17H20 M6.5 2H20v20H6.5A2.5 2.5 0 0 1 4 19.5v-15A2.5 2.5 0 0 1 6.5 2z",
  apple:   ["M12 2a3 3 0 0 0-3 3", "M8 6c-2.5.5-4 2.5-4 5.5 0 4 3 8 5 8 1 0 1.5-.5 3-.5s2 .5 3 .5c2 0 5-4 5-8 0-3-1.5-5-4-5.5"],
  dumbbell:"M6.5 6.5h11M6.5 17.5h11M3 9.5h3M18 9.5h3M3 14.5h3M18 14.5h3M6 6v12M18 6v12",
  star:    "M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z",
  plus:    "M12 5v14M5 12h14",
  check:   "M20 6L9 17l-5-5",
  user:    ["M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2", "M12 11a4 4 0 1 0 0-8 4 4 0 0 0 0 8z"],
  logout:  ["M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4", "M16 17l5-5-5-5", "M21 12H9"],
  flame:   "M8.5 14.5A2.5 2.5 0 0 0 11 12c0-1.38-.5-2-1-3-1.072-2.143-.224-4.054 2-6 .5 2.5 2 4.9 4 6.5 2 1.6 3 3.5 3 5.5a7 7 0 1 1-14 0c0-1.153.433-2.294 1-3a2.5 2.5 0 0 0 2.5 3z",
  sparkle: ["M12 3l1.5 4.5L18 9l-4.5 1.5L12 15l-1.5-4.5L6 9l4.5-1.5z", "M19 3l.75 2.25L22 6l-2.25.75L19 9l-.75-2.25L16 6l2.25-.75z"],
  refresh: "M23 4v6h-6 M1 20v-6h6 M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15",
  x:       "M18 6L6 18M6 6l12 12",
  download:"M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4 M7 10l5 5 5-5 M12 15V3",
  timer:   ["M12 2a10 10 0 1 0 0 20 10 10 0 0 0 0-20z", "M12 6v6l4 2"],
  play:    "M5 3l14 9-14 9V3z",
  pause:   "M6 4h4v16H6z M14 4h4v16h-4z",
};

// ─── AUTH ─────────────────────────────────────────────────────────────────────
function AuthScreen({ onAuth }) {
  const [mode, setMode] = useState("login");
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [pw, setPw] = useState("");
  const [tz] = useState(Intl.DateTimeFormat().resolvedOptions().timeZone);
  const [runGoal, setRunGoal] = useState("3");
  const [workoutGoal, setWorkoutGoal] = useState("4");
  const [calGoal, setCalGoal] = useState("2000");
  const [err, setErr] = useState("");

  const submit = () => {
    setErr("");
    if (!email.trim() || !pw.trim()) return setErr("Email and password required.");
    const key = `forge_u_${email.toLowerCase().trim()}`;
    if (mode === "register") {
      if (!name.trim()) return setErr("Name required.");
      if (pw.length < 6) return setErr("Password must be 6+ characters.");
      if (localStorage.getItem(key)) return setErr("Account already exists.");
      const u = { name: name.trim(), email: email.toLowerCase().trim(), tz, joined: TODAY(), goals: { runs: parseInt(runGoal) || 3, workouts: parseInt(workoutGoal) || 4, calories: parseInt(calGoal) || 2000 } };
      localStorage.setItem(key, JSON.stringify({ ...u, pw }));
      onAuth(u);
    } else {
      const raw = localStorage.getItem(key);
      if (!raw) return setErr("No account found.");
      const s = JSON.parse(raw);
      if (s.pw !== pw) return setErr("Incorrect password.");
      onAuth({ name: s.name, email: s.email, tz: s.tz, joined: s.joined, goals: s.goals });
    }
  };

  return (
    <div className="auth-wrap">
      <div className="auth-bg" />
      <div className="auth-grid" />
      <div className="auth-logo-wrap">
        <div className="forge-logo">FORGE</div>
        <div className="forge-sub">Build discipline. Stay consistent.</div>
      </div>
      <div className="auth-content">
        <div className="auth-card">
          <div className="auth-title">{mode === "login" ? "Welcome back" : "Create account"}</div>
          {mode === "register" && (
            <div className="field"><label>Your Name</label><input className="inp" placeholder="Alex" value={name} onChange={e => setName(e.target.value)} /></div>
          )}
          <div className="field"><label>Email</label><input className="inp" type="email" placeholder="you@email.com" value={email} onChange={e => setEmail(e.target.value)} /></div>
          <div className="field"><label>Password</label><input className="inp" type="password" placeholder="••••••••" value={pw} onChange={e => setPw(e.target.value)} /></div>
          {mode === "register" && (
            <>
              <div className="field"><label>Timezone</label><input className="inp" value={tz} readOnly style={{ opacity: 0.6 }} /></div>
              <div className="row">
                <div className="col field"><label>Runs/week</label><input className="inp" type="number" value={runGoal} onChange={e => setRunGoal(e.target.value)} /></div>
                <div className="col field"><label>Workouts/wk</label><input className="inp" type="number" value={workoutGoal} onChange={e => setWorkoutGoal(e.target.value)} /></div>
                <div className="col field"><label>Calorie goal</label><input className="inp" type="number" value={calGoal} onChange={e => setCalGoal(e.target.value)} /></div>
              </div>
            </>
          )}
          {err && <div className="err-msg">{err}</div>}
          <button className="btn" onClick={submit}>{mode === "login" ? "Sign In" : "Create Account"}</button>
          <div className="auth-toggle">
            {mode === "login" ? <>No account? <span onClick={() => { setMode("register"); setErr(""); }}>Register free</span></> : <>Have account? <span onClick={() => { setMode("login"); setErr(""); }}>Sign in</span></>}
          </div>
        </div>
      </div>
    </div>
  );
}

// ─── BOTTOM NAV ───────────────────────────────────────────────────────────────
function BottomNav({ tab, setTab }) {
  const items = [
    { id: "home",     icon: Icons.home,     label: "Home" },
    { id: "run",      icon: Icons.run,      label: "Run" },
    { id: "focus",    icon: Icons.timer,    label: "Focus" },
    { id: "calories", icon: Icons.apple,    label: "Calories" },
    { id: "workout",  icon: Icons.dumbbell, label: "Workout" },
  ];
  return (
    <nav className="nav">
      {items.map(i => (
        <div key={i.id} className={`nav-item ${tab === i.id ? "active" : ""}`} onClick={() => setTab(i.id)}>
          <Ic d={i.icon} size={20} />
          <span className="nav-label">{i.label}</span>
          <div className="nav-pip" />
        </div>
      ))}
    </nav>
  );
}

// ─── DASHBOARD ────────────────────────────────────────────────────────────────
function HomeScreen({ user, runs, focusSessions, meals, workouts, weeklyReview, setWeeklyReview }) {
  const today = TODAY();
  const todayRun = runs.some(r => r.date === today);
  const todayFocus = focusSessions.filter(f => f.date === today).reduce((s, f) => s + f.minutes, 0);
  const todayCalories = meals.filter(m => m.date === today).reduce((s, m) => s + m.calories, 0);
  const todayWorkout = workouts.some(w => w.date === today && w.completed);

  const runStreak = calcStreak(runs.map(r => r.date));
  const focusStreak = calcStreak([...new Set(focusSessions.map(f => f.date))]);
  const mealStreak = calcStreak([...new Set(meals.map(m => m.date))]);
  const workoutStreak = calcStreak(workouts.filter(w => w.completed).map(w => w.date));

  const overallScore = Math.round([todayRun, todayFocus >= 25, todayCalories > 0, todayWorkout].filter(Boolean).length / 4 * 100);

  const weekDates = getWeekDates();
  const allActivity = weekDates.map(d => {
    const score = [
      runs.some(r => r.date === d),
      focusSessions.some(f => f.date === d),
      meals.some(m => m.date === d),
      workouts.some(w => w.date === d && w.completed),
    ].filter(Boolean).length / 4;
    return { date: d, score };
  });
  const maxScore = Math.max(...allActivity.map(x => x.score), 0.01);

  const dateStr = new Date().toLocaleDateString("en-US", { weekday: "long", month: "short", day: "numeric" });

  return (
    <div className="screen">
      <div className="screen-hd">
        <div style={{ fontSize: 13, color: "var(--muted2)", marginBottom: 4 }}>{dateStr}</div>
        <div className="screen-title">Good {new Date().getHours() < 12 ? "morning" : new Date().getHours() < 17 ? "afternoon" : "evening"},</div>
        <div style={{ fontSize: 28, fontWeight: 700, color: "var(--amber)" }}>{user.name.split(" ")[0]}</div>
      </div>

      {/* Today's Score */}
      <div className="summary-card" style={{ marginBottom: 16 }}>
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 14 }}>
          <span className="label">Today's Discipline</span>
          <span className="mono" style={{ fontSize: 24, fontWeight: 700, color: "var(--amber)" }}>{overallScore}%</span>
        </div>
        <div className="pbar" style={{ height: 8 }}><div className="pbar-fill" style={{ width: `${overallScore}%` }} /></div>
      </div>

      {/* Checklist */}
      <div className="card">
        <div className="card-hd"><span className="card-hd-title">Today's Checklist</span></div>
        {[
          { icon: "🏃", label: "Run", done: todayRun, value: todayRun ? "Done" : "Not done" },
          { icon: "📚", label: "Focus", done: todayFocus >= 25, value: `${todayFocus} min` },
          { icon: "🍎", label: "Calories", done: todayCalories > 0, value: todayCalories > 0 ? `${todayCalories} kcal` : "Not logged" },
          { icon: "💪", label: "Workout", done: todayWorkout, value: todayWorkout ? "Done" : "Not done" },
        ].map((item, i) => (
          <div key={i} className="checklist-item">
            <div className={`check-circle ${item.done ? "done" : ""}`}>
              {item.done ? <span className="check-icon">✓</span> : <span style={{ fontSize: 10, color: "var(--muted)" }}>{item.icon}</span>}
            </div>
            <span className="checklist-label">{item.label}</span>
            <span className="checklist-value" style={{ color: item.done ? "var(--amber)" : "var(--muted)" }}>{item.value}</span>
          </div>
        ))}
      </div>

      {/* Streaks */}
      <div className="stat-grid">
        <div className="stat-box">
          <div className="stat-num">{Math.max(runStreak, focusStreak, mealStreak, workoutStreak)}</div>
          <div className="stat-lbl">Best Streak 🔥</div>
        </div>
        <div className="stat-box">
          <div className="stat-num">{weeklyScore([...runs.map(r => r.date), ...focusSessions.map(f => f.date), ...workouts.filter(w => w.completed).map(w => w.date)])}%</div>
          <div className="stat-lbl">Week Score</div>
          <div className="pbar" style={{ marginTop: 8 }}>
            <div className="pbar-fill" style={{ width: `${weeklyScore([...runs.map(r => r.date), ...focusSessions.map(f => f.date), ...workouts.filter(w => w.completed).map(w => w.date)])}%` }} />
          </div>
        </div>
      </div>

      {/* Weekly Chart */}
      <div className="card">
        <div className="card-hd"><span className="card-hd-title">Weekly Activity</span></div>
        <div className="week-bars">
          {allActivity.map((d, i) => {
            const isToday = d.date === today;
            const h = Math.max((d.score / maxScore) * 50, d.score > 0 ? 6 : 3);
            return (
              <div key={i} className="wb-col">
                <div className={`wb-bar ${d.score > 0 ? (isToday ? "today" : "on") : ""}`} style={{ height: h }} />
                <span className="wb-day">{getDayOfWeek(d.date)}</span>
              </div>
            );
          })}
        </div>
      </div>

      {/* Weekly Review Preview */}
      {weeklyReview && (
        <div className="summary-card">
          <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 10 }}>
            <Ic d={Icons.sparkle} size={16} />
            <span className="label" style={{ color: "var(--amber)" }}>Latest Weekly Review</span>
          </div>
          <div style={{ fontSize: 13, color: "var(--muted2)", lineHeight: 1.7 }}>{weeklyReview.summary?.slice(0, 130)}…</div>
        </div>
      )}
    </div>
  );
}

// ─── RUN TRACKER ──────────────────────────────────────────────────────────────
function RunScreen({ runs, setRuns, user, showToast }) {
  const [running, setRunning] = useState(false);
  const [elapsed, setElapsed] = useState(0);
  const [distance, setDistance] = useState(0);
  const [showSummary, setShowSummary] = useState(null);
  const timerRef = useRef(null);
  const startTimeRef = useRef(null);

  // Simulated distance (since GPS requires native)
  const distRef = useRef(0);

  useEffect(() => {
    if (running) {
      startTimeRef.current = Date.now() - elapsed * 1000;
      timerRef.current = setInterval(() => {
        const secs = Math.floor((Date.now() - startTimeRef.current) / 1000);
        setElapsed(secs);
        // Simulate distance: ~8km/h pace → ~2.22m/s
        distRef.current += 2.22;
        setDistance(Math.floor(distRef.current));
      }, 1000);
    }
    return () => clearInterval(timerRef.current);
  }, [running]);

  const startRun = () => {
    setElapsed(0);
    setDistance(0);
    distRef.current = 0;
    setRunning(true);
  };

  const stopRun = () => {
    clearInterval(timerRef.current);
    setRunning(false);
    const summary = { id: Date.now(), date: TODAY(), duration: elapsed, distance, note: "" };
    setShowSummary(summary);
  };

  const saveRun = (note) => {
    const run = { ...showSummary, note };
    setRuns(prev => [...prev, run]);
    setShowSummary(null);
    setElapsed(0);
    setDistance(0);
    showToast("Run saved! 🏃");
  };

  const today = TODAY();
  const weekDates = getWeekDates();
  const runStreak = calcStreak(runs.map(r => r.date));
  const weekRuns = runs.filter(r => weekDates.includes(r.date)).length;
  const goal = user.goals?.runs || 3;

  return (
    <div className="screen">
      <div className="screen-hd">
        <div className="screen-title">Run Tracker</div>
        <div className="screen-sub">Outdoor running · GPS-ready</div>
      </div>

      {/* Stats Row */}
      <div className="stat-grid">
        <div className="stat-box">
          <div className="stat-num">{runStreak}</div>
          <div className="stat-lbl">Day Streak 🔥</div>
        </div>
        <div className="stat-box">
          <div className="stat-num">{weekRuns}/{goal}</div>
          <div className="stat-lbl">Runs This Week</div>
          <div className="pbar" style={{ marginTop: 8 }}><div className="pbar-fill" style={{ width: `${Math.min((weekRuns / goal) * 100, 100)}%` }} /></div>
        </div>
      </div>

      {/* Live Timer */}
      {running && (
        <div className="live-display">
          <div className="live-time">{formatTime(elapsed)}</div>
          <div className="live-sub">{formatDist(distance)}</div>
          <div className="live-label">Distance (simulated)</div>
        </div>
      )}

      {!running && !showSummary && (
        <div style={{ textAlign: "center", padding: "20px 0 10px" }}>
          <div style={{ fontSize: 13, color: "var(--muted2)", marginBottom: 20 }}>
            {runs.filter(r => r.date === today).length > 0 ? "✓ Already ran today! Run again?" : "Ready to run?"}
          </div>
        </div>
      )}

      {/* Big Button */}
      {!showSummary && (
        <div className="big-btn-wrap">
          <button className={`big-btn ${running ? "active" : ""}`} onClick={running ? stopRun : startRun}>
            <div className="big-btn-icon">{running ? "⏹" : "▶"}</div>
            <div>{running ? "Stop Run" : "Start Run"}</div>
          </button>
        </div>
      )}

      {/* Post-Run Summary */}
      {showSummary && <RunSummaryCard summary={showSummary} onSave={saveRun} onDiscard={() => setShowSummary(null)} />}

      {/* Weekly Progress Bar */}
      <div className="card">
        <div className="card-hd">
          <span className="card-hd-title">This Week</span>
          <span className="streak-badge">🔥 {runStreak} day streak</span>
        </div>
        <div className="wdots">
          {weekDates.map((d, i) => {
            const hasRun = runs.some(r => r.date === d);
            return <div key={i} className={`wdot ${hasRun ? (d === today ? "today" : "on") : ""}`} />;
          })}
        </div>
        <div style={{ display: "flex", justifyContent: "space-between", marginTop: 8 }}>
          {weekDates.map((d, i) => <span key={i} className="wb-day">{getDayOfWeek(d)}</span>)}
        </div>
      </div>

      {/* Run History */}
      {runs.length > 0 && (
        <>
          <div className="section-hd"><span className="label">Recent Runs</span></div>
          {[...runs].reverse().slice(0, 5).map(r => (
            <div key={r.id} className="list-item">
              <div className="list-item-left">
                <div className="list-item-name">{formatDist(r.distance)}</div>
                <div className="list-item-meta">{r.date} · {formatTime(r.duration)}</div>
                {r.note && <div className="list-item-meta" style={{ fontStyle: "italic", marginTop: 2 }}>{r.note}</div>}
              </div>
              <div className="list-item-right">
                <span className="badge badge-amber">{Math.round((r.distance / 1000) / (r.duration / 3600) * 10) / 10} km/h</span>
              </div>
            </div>
          ))}
        </>
      )}
    </div>
  );
}

function RunSummaryCard({ summary, onSave, onDiscard }) {
  const [note, setNote] = useState("");
  return (
    <div className="summary-card" style={{ margin: "20px 0" }}>
      <div style={{ display: "flex", alignItems: "center", gap: 10, marginBottom: 16 }}>
        <span style={{ fontSize: 32 }}>🏁</span>
        <div>
          <div style={{ fontSize: 18, fontWeight: 700 }}>Run Complete!</div>
          <div style={{ fontSize: 12, color: "var(--muted2)" }}>{TODAY()}</div>
        </div>
      </div>
      <div className="stat-grid" style={{ marginBottom: 16 }}>
        <div className="stat-box"><div className="stat-num" style={{ fontSize: 28 }}>{formatTime(summary.duration)}</div><div className="stat-lbl">Duration</div></div>
        <div className="stat-box"><div className="stat-num" style={{ fontSize: 28 }}>{formatDist(summary.distance)}</div><div className="stat-lbl">Distance</div></div>
      </div>
      <div className="field" style={{ marginBottom: 14 }}>
        <label style={{ display: "block", fontSize: 11, color: "var(--muted2)", letterSpacing: "1px", textTransform: "uppercase", marginBottom: 6, fontWeight: 600 }}>Note (optional)</label>
        <input className="inp" placeholder="How did it feel?" value={note} onChange={e => setNote(e.target.value)} />
      </div>
      <div className="row">
        <button className="btn" onClick={() => onSave(note)}>Save Run</button>
        <button className="btn btn-ghost btn-sm" style={{ width: "auto", padding: "0 16px" }} onClick={onDiscard}>Discard</button>
      </div>
    </div>
  );
}

// ─── FOCUS TIMER ──────────────────────────────────────────────────────────────
function FocusScreen({ focusSessions, setFocusSessions, showToast }) {
  const PRESETS = [25, 50];
  const [duration, setDuration] = useState(25);
  const [custom, setCustom] = useState("");
  const [running, setRunning] = useState(false);
  const [paused, setPaused] = useState(false);
  const [remaining, setRemaining] = useState(25 * 60);
  const [showNudge, setShowNudge] = useState(false);
  const timerRef = useRef(null);
  const totalSecs = duration * 60;

  useEffect(() => {
    const handleVisibility = () => {
      if (running && !paused && document.hidden) {
        setPaused(true);
        clearInterval(timerRef.current);
        setShowNudge(true);
      }
    };
    document.addEventListener("visibilitychange", handleVisibility);
    return () => document.removeEventListener("visibilitychange", handleVisibility);
  }, [running, paused]);

  useEffect(() => {
    if (running && !paused) {
      timerRef.current = setInterval(() => {
        setRemaining(r => {
          if (r <= 1) {
            clearInterval(timerRef.current);
            completeSession(duration);
            return 0;
          }
          return r - 1;
        });
      }, 1000);
    } else {
      clearInterval(timerRef.current);
    }
    return () => clearInterval(timerRef.current);
  }, [running, paused]);

  const completeSession = (mins) => {
    setRunning(false);
    setPaused(false);
    const session = { id: Date.now(), date: TODAY(), minutes: mins };
    setFocusSessions(prev => [...prev, session]);
    showToast(`Focus session complete! +${mins} min 📚`);
    setRemaining(duration * 60);
  };

  const startSession = (mins) => {
    const m = parseInt(custom) || mins;
    setDuration(m);
    setRemaining(m * 60);
    setRunning(true);
    setPaused(false);
    setShowNudge(false);
  };

  const pauseResume = () => {
    setPaused(p => !p);
    setShowNudge(false);
  };

  const stopSession = () => {
    clearInterval(timerRef.current);
    setRunning(false);
    setPaused(false);
    setShowNudge(false);
    // Save partial — only if >5 minutes done
    const done = totalSecs - remaining;
    if (done >= 300) {
      const partialMins = Math.floor(done / 60);
      const session = { id: Date.now(), date: TODAY(), minutes: partialMins, partial: true };
      setFocusSessions(prev => [...prev, session]);
      showToast(`Partial session saved: ${partialMins} min`);
    }
    setRemaining(duration * 60);
  };

  const today = TODAY();
  const todayMins = focusSessions.filter(f => f.date === today).reduce((s, f) => s + f.minutes, 0);
  const weekDates = getWeekDates();
  const focusDates = [...new Set(focusSessions.map(f => f.date))];
  const focusStreak = calcStreak(focusDates);
  const weekMins = focusSessions.filter(f => weekDates.includes(f.date)).reduce((s, f) => s + f.minutes, 0);

  const progress = running ? (remaining / totalSecs) : 1;
  const r = 80, circ = 2 * Math.PI * r;
  const dash = circ * progress;

  return (
    <div className="screen">
      <div className="screen-hd">
        <div className="screen-title">Focus Timer</div>
        <div className="screen-sub">Deep work · distraction-limited</div>
      </div>

      {showNudge && (
        <div className="nudge-banner">
          ⚠️ You left the app — session paused. Return to continue.
          <button className="btn btn-sm" style={{ marginTop: 8, background: "none", border: "none", color: "var(--amber)", padding: "4px 0", fontSize: 13, cursor: "pointer", width: "auto" }} onClick={() => { setShowNudge(false); setPaused(false); }}>
            Resume →
          </button>
        </div>
      )}

      {/* Duration Presets */}
      {!running && (
        <div style={{ marginBottom: 24 }}>
          <div className="label" style={{ marginBottom: 10 }}>Session Length</div>
          <div className="tags">
            {PRESETS.map(p => <span key={p} className={`tag ${duration === p && !custom ? "on" : ""}`} onClick={() => { setDuration(p); setRemaining(p * 60); setCustom(""); }}>{p} min</span>)}
            <input className="inp" type="number" placeholder="Custom min" value={custom} onChange={e => { setCustom(e.target.value); if (e.target.value) setDuration(parseInt(e.target.value) || 25); }} style={{ width: 110, padding: "8px 12px", fontSize: 13 }} />
          </div>
        </div>
      )}

      {/* Arc Timer */}
      <div className="focus-ring" style={{ marginBottom: 24 }}>
        <svg width={180} height={180} viewBox="0 0 180 180" style={{ position: "absolute", inset: 0, transform: "rotate(-90deg)" }}>
          <circle cx={90} cy={90} r={r} fill="none" stroke="var(--s3)" strokeWidth={8} />
          <circle cx={90} cy={90} r={r} fill="none" stroke="var(--amber)" strokeWidth={8}
            strokeDasharray={`${dash} ${circ}`} strokeLinecap="round"
            style={{ transition: "stroke-dasharray 1s linear" }} />
        </svg>
        <div className="focus-center">
          <div className="live-time" style={{ fontSize: 40 }}>{formatTime(remaining)}</div>
          {running && <div style={{ fontSize: 11, color: "var(--muted)", letterSpacing: 2, textTransform: "uppercase", marginTop: 4 }}>{paused ? "PAUSED" : "FOCUSING"}</div>}
        </div>
      </div>

      {/* Controls */}
      {!running ? (
        <div className="big-btn-wrap" style={{ marginTop: 0 }}>
          <button className="big-btn" onClick={() => startSession(duration)}>
            <div className="big-btn-icon">📚</div>
            <div>Start Focus</div>
          </button>
        </div>
      ) : (
        <div className="row" style={{ justifyContent: "center", gap: 12, marginBottom: 24 }}>
          <button className="btn btn-outline btn-sm" style={{ minWidth: 120 }} onClick={pauseResume}>
            {paused ? "▶ Resume" : "⏸ Pause"}
          </button>
          <button className="btn btn-ghost btn-sm" style={{ minWidth: 100 }} onClick={stopSession}>
            ⏹ Stop
          </button>
        </div>
      )}

      {/* Stats */}
      <div className="stat-grid">
        <div className="stat-box">
          <div className="stat-num">{todayMins}</div>
          <div className="stat-lbl">Min Today</div>
        </div>
        <div className="stat-box">
          <div className="stat-num">{focusStreak}</div>
          <div className="stat-lbl">Day Streak 🔥</div>
        </div>
      </div>

      <div className="card">
        <div className="card-hd">
          <span className="card-hd-title">This Week</span>
          <span className="mono" style={{ fontSize: 13, color: "var(--amber)" }}>{weekMins} min</span>
        </div>
        <div className="wdots">
          {weekDates.map((d, i) => {
            const mins = focusSessions.filter(f => f.date === d).reduce((s, f) => s + f.minutes, 0);
            return <div key={i} className={`wdot ${mins > 0 ? (d === today ? "today" : "on") : ""}`} />;
          })}
        </div>
        <div style={{ display: "flex", justifyContent: "space-between", marginTop: 8 }}>
          {weekDates.map((d, i) => <span key={i} className="wb-day">{getDayOfWeek(d)}</span>)}
        </div>
      </div>

      {/* Recent Sessions */}
      {focusSessions.length > 0 && (
        <>
          <div className="section-hd"><span className="label">Recent Sessions</span></div>
          {[...focusSessions].reverse().slice(0, 4).map(s => (
            <div key={s.id} className="list-item">
              <div className="list-item-left">
                <div className="list-item-name">{s.minutes} min focus</div>
                <div className="list-item-meta">{s.date}{s.partial ? " · partial" : ""}</div>
              </div>
              <span className={`badge ${s.partial ? "badge-blue" : "badge-amber"}`}>{s.partial ? "Partial" : "Complete"}</span>
            </div>
          ))}
        </>
      )}
    </div>
  );
}

// ─── CALORIE TRACKER ─────────────────────────────────────────────────────────
function CalorieScreen({ meals, setMeals, user, setModal, showToast }) {
  const today = TODAY();
  const goal = user.goals?.calories || 2000;
  const todayMeals = meals.filter(m => m.date === today);
  const todayTotal = todayMeals.reduce((s, m) => s + m.calories, 0);
  const pct = Math.min((todayTotal / goal) * 100, 100);
  const over = todayTotal > goal;

  const weekDates = getWeekDates();
  const mealDates = [...new Set(meals.map(m => m.date))];
  const streak = calcStreak(mealDates);
  const weekAvg = Math.round(weekDates.map(d => meals.filter(m => m.date === d).reduce((s, m) => s + m.calories, 0)).reduce((a, b) => a + b, 0) / 7);

  const deleteMeal = (id) => {
    setMeals(prev => prev.filter(m => m.id !== id));
    showToast("Meal removed");
  };

  return (
    <div className="screen">
      <div className="screen-hd">
        <div className="screen-title">Calories</div>
        <div className="screen-sub">Simple daily calorie awareness</div>
      </div>

      {/* Today's Total */}
      <div className="summary-card" style={{ marginBottom: 12 }}>
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "baseline", marginBottom: 10 }}>
          <div>
            <div className="mono" style={{ fontSize: 48, fontWeight: 700, color: over ? "var(--danger)" : "var(--amber)", lineHeight: 1 }}>{todayTotal}</div>
            <div className="label" style={{ marginTop: 4 }}>kcal today</div>
          </div>
          <div style={{ textAlign: "right" }}>
            <div className="mono" style={{ fontSize: 18, color: "var(--muted2)" }}>/ {goal}</div>
            <div className="label">daily goal</div>
          </div>
        </div>
        <div className="pbar" style={{ height: 8 }}>
          <div className={`pbar-fill ${over ? "danger" : pct > 85 ? "" : "green"}`} style={{ width: `${pct}%` }} />
        </div>
        {over && <div style={{ fontSize: 12, color: "var(--danger)", marginTop: 8 }}>+{todayTotal - goal} kcal over goal</div>}
      </div>

      <div className="stat-grid">
        <div className="stat-box">
          <div className="stat-num">{streak}</div>
          <div className="stat-lbl">Day Streak 🔥</div>
        </div>
        <div className="stat-box">
          <div className="stat-num">{weekAvg}</div>
          <div className="stat-lbl">Weekly Avg</div>
        </div>
      </div>

      {/* Add Meal */}
      <button className="btn" style={{ marginBottom: 16 }} onClick={() => setModal("add_meal")}>+ Add Meal</button>

      {/* Today's Meals */}
      {todayMeals.length === 0 ? (
        <div className="empty"><div className="empty-icon">🍽️</div>No meals logged today</div>
      ) : (
        <>
          <div className="section-hd"><span className="label">Today's Meals</span><span className="mono" style={{ fontSize: 13, color: "var(--muted2)" }}>{todayMeals.length} entries</span></div>
          {todayMeals.map(m => (
            <div key={m.id} className="list-item">
              <div className="list-item-left">
                <div className="list-item-name">{m.name}</div>
                <div className="list-item-meta">{m.mealType} · {m.time}</div>
              </div>
              <div className="list-item-right">
                <span className="mono" style={{ fontSize: 16, fontWeight: 700, color: "var(--amber)" }}>{m.calories}</span>
                <span className="label">kcal</span>
                <button onClick={() => deleteMeal(m.id)} style={{ background: "none", border: "none", color: "var(--muted)", fontSize: 11, cursor: "pointer" }}>remove</button>
              </div>
            </div>
          ))}
        </>
      )}

      {/* Weekly Summary */}
      <div className="card" style={{ marginTop: 16 }}>
        <div className="card-hd"><span className="card-hd-title">Weekly Calories</span></div>
        <div className="week-bars">
          {weekDates.map((d, i) => {
            const dayTotal = meals.filter(m => m.date === d).reduce((s, m) => s + m.calories, 0);
            const maxCal = Math.max(...weekDates.map(wd => meals.filter(m => m.date === wd).reduce((s, m) => s + m.calories, 0)), goal);
            const h = Math.max((dayTotal / maxCal) * 50, dayTotal > 0 ? 6 : 3);
            return (
              <div key={i} className="wb-col">
                <div className={`wb-bar ${dayTotal > 0 ? (d === today ? "today" : "on") : ""}`} style={{ height: h }} />
                <span className="wb-day">{getDayOfWeek(d)}</span>
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}

// ─── WORKOUT TRACKER ─────────────────────────────────────────────────────────
function WorkoutScreen({ workouts, setWorkouts, user, setModal, showToast }) {
  const today = TODAY();
  const goal = user.goals?.workouts || 4;
  const weekDates = getWeekDates();
  const completedDates = workouts.filter(w => w.completed).map(w => w.date);
  const streak = calcStreak(completedDates);
  const weekDone = workouts.filter(w => w.completed && weekDates.includes(w.date)).length;

  const todayWorkout = workouts.find(w => w.date === today && !w.completed);
  const todayDone = workouts.find(w => w.date === today && w.completed);

  const markComplete = (id) => {
    setWorkouts(prev => prev.map(w => w.id === id ? { ...w, completed: true, completedAt: new Date().toISOString() } : w));
    showToast("Workout complete! 💪");
  };

  const addQuickLog = () => {
    const quick = { id: Date.now(), date: today, name: "Quick Workout", type: "General", completed: true, duration: 0, completedAt: new Date().toISOString() };
    setWorkouts(prev => [...prev, quick]);
    showToast("Workout logged! 💪");
  };

  return (
    <div className="screen">
      <div className="screen-hd">
        <div className="screen-title">Workouts</div>
        <div className="screen-sub">Strength · consistency · progress</div>
      </div>

      <div className="stat-grid">
        <div className="stat-box">
          <div className="stat-num">{streak}</div>
          <div className="stat-lbl">Day Streak 🔥</div>
        </div>
        <div className="stat-box">
          <div className="stat-num">{weekDone}/{goal}</div>
          <div className="stat-lbl">This Week</div>
          <div className="pbar" style={{ marginTop: 8 }}><div className="pbar-fill green" style={{ width: `${Math.min((weekDone / goal) * 100, 100)}%` }} /></div>
        </div>
      </div>

      {/* Today's Workout */}
      <div className="card" style={{ borderColor: todayDone ? "rgba(16,185,129,0.3)" : todayWorkout ? "rgba(245,158,11,0.3)" : "var(--border)" }}>
        <div className="card-hd"><span className="card-hd-title">Today</span></div>
        {todayDone ? (
          <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
            <span style={{ fontSize: 28 }}>✅</span>
            <div>
              <div style={{ fontSize: 16, fontWeight: 600, color: "var(--green)" }}>{todayDone.name}</div>
              <div className="caption">{todayDone.type} · Complete</div>
            </div>
          </div>
        ) : todayWorkout ? (
          <>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 12 }}>
              <div>
                <div style={{ fontSize: 16, fontWeight: 600 }}>{todayWorkout.name}</div>
                <div className="caption">{todayWorkout.type}</div>
              </div>
              <span className="badge badge-amber">Planned</span>
            </div>
            <button className="btn btn-green" onClick={() => markComplete(todayWorkout.id)}>✓ Mark as Completed</button>
          </>
        ) : (
          <div style={{ display: "flex", gap: 10 }}>
            <button className="btn" onClick={() => setModal("plan_workout")} style={{ flex: 1 }}>Plan Workout</button>
            <button className="btn btn-outline" onClick={addQuickLog} style={{ flex: 1 }}>Quick Log</button>
          </div>
        )}
      </div>

      {/* Week Dots */}
      <div className="card">
        <div className="card-hd"><span className="card-hd-title">This Week</span><span className="streak-badge">🔥 {streak} streak</span></div>
        <div className="wdots">
          {weekDates.map((d, i) => {
            const done = workouts.some(w => w.date === d && w.completed);
            return <div key={i} className={`wdot ${done ? (d === today ? "today" : "on") : ""}`} />;
          })}
        </div>
        <div style={{ display: "flex", justifyContent: "space-between", marginTop: 8 }}>
          {weekDates.map((d, i) => <span key={i} className="wb-day">{getDayOfWeek(d)}</span>)}
        </div>
      </div>

      {/* History */}
      {workouts.filter(w => w.completed).length > 0 && (
        <>
          <div className="section-hd"><span className="label">Workout History</span><button className="btn btn-ghost btn-sm" onClick={() => setModal("plan_workout")}>+ Plan</button></div>
          {[...workouts].filter(w => w.completed).reverse().slice(0, 6).map(w => (
            <div key={w.id} className="list-item">
              <div className="list-item-left">
                <div className="list-item-name">{w.name}</div>
                <div className="list-item-meta">{w.date} · {w.type}{w.duration ? ` · ${w.duration} min` : ""}</div>
              </div>
              <span className="badge badge-green">Done</span>
            </div>
          ))}
        </>
      )}
    </div>
  );
}

// ─── WEEKLY REVIEW ────────────────────────────────────────────────────────────
function ReviewScreen({ user, runs, focusSessions, meals, workouts, weeklyReview, setWeeklyReview }) {
  const [loading, setLoading] = useState(false);

  const canGenerate = !weeklyReview || daysSince(weeklyReview.generatedOn) >= 7;

  const buildPayload = () => {
    const weekDates = getWeekDates();
    const runCount = runs.filter(r => weekDates.includes(r.date)).length;
    const focusMins = focusSessions.filter(f => weekDates.includes(f.date)).reduce((s, f) => s + f.minutes, 0);
    const avgCal = Math.round(weekDates.map(d => meals.filter(m => m.date === d).reduce((s, m) => s + m.calories, 0)).reduce((a, b) => a + b, 0) / 7);
    const workoutCount = workouts.filter(w => w.completed && weekDates.includes(w.date)).length;
    const runStreak = calcStreak(runs.map(r => r.date));
    const focusStreak = calcStreak([...new Set(focusSessions.map(f => f.date))]);
    return `User: ${user.name}. Goals: ${user.goals?.runs} runs/wk, ${user.goals?.workouts} workouts/wk, ${user.goals?.calories} kcal/day. This week: ${runCount} runs (goal: ${user.goals?.runs}), ${focusMins} focus min, avg ${avgCal} kcal/day (goal: ${user.goals?.calories}), ${workoutCount} workouts (goal: ${user.goals?.workouts}). Best streaks: run ${runStreak}d, focus ${focusStreak}d.`;
  };

  const generate = async () => {
    setLoading(true);
    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          system: `You are a calm, disciplined fitness and productivity coach. Given weekly tracking data, return a JSON object:
{ "summary": "2-3 sentence paragraph of what went well and where they slipped", "suggestions": [{"title": "...", "text": "1 sentence"}] (exactly 2-3), "planAdjustment": "1-2 sentence practical tweak for next week" }
Be concise, actionable, supportive. No medical advice. No fluff. Return only valid JSON.`,
          messages: [{ role: "user", content: buildPayload() }],
        }),
      });
      const data = await res.json();
      const raw = data.content?.[0]?.text || "{}";
      const parsed = JSON.parse(raw.replace(/```json|```/g, "").trim());
      setWeeklyReview({ ...parsed, generatedOn: TODAY(), planAccepted: false });
    } catch {
      setWeeklyReview({ summary: "Unable to connect. Check your internet and try again.", suggestions: [], planAdjustment: null, generatedOn: TODAY(), planAccepted: false });
    }
    setLoading(false);
  };

  const exportCSV = () => {
    const lines = ["Type,Date,Value,Detail"];
    runs.forEach(r => lines.push(`Run,${r.date},${formatDist(r.distance)},${formatTime(r.duration)}`));
    focusSessions.forEach(f => lines.push(`Focus,${f.date},${f.minutes} min,${f.partial ? "partial" : "complete"}`));
    meals.forEach(m => lines.push(`Meal,${m.date},${m.calories} kcal,${m.name}`));
    workouts.filter(w => w.completed).forEach(w => lines.push(`Workout,${w.date},${w.name},${w.type}`));
    const blob = new Blob([lines.join("\n")], { type: "text/csv" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a"); a.href = url; a.download = "forge-data.csv"; a.click();
  };

  return (
    <div className="screen">
      <div className="screen-hd">
        <div className="screen-title">Weekly Review</div>
        <div className="screen-sub">AI-powered · once per week</div>
      </div>

      {!weeklyReview && !loading && (
        <div className="card" style={{ textAlign: "center", padding: "36px 20px" }}>
          <div style={{ fontSize: 44, marginBottom: 12 }}>✨</div>
          <div style={{ fontSize: 17, fontWeight: 700, marginBottom: 8 }}>Ready for your weekly review?</div>
          <div style={{ fontSize: 13, color: "var(--muted2)", marginBottom: 24, lineHeight: 1.7 }}>
            Your AI coach will analyze your last 7 days across all four modules and give you honest, actionable feedback.
          </div>
          <button className="btn" onClick={generate}>Generate Review</button>
        </div>
      )}

      {loading && (
        <div className="card">
          <div style={{ display: "flex", alignItems: "center", gap: 14, padding: "16px 0" }}>
            <div style={{ width: 24, height: 24, border: "2px solid var(--s3)", borderTopColor: "var(--amber)", borderRadius: "50%", animation: "spin 0.8s linear infinite", flexShrink: 0 }} />
            <span style={{ color: "var(--muted2)", fontSize: 14 }}>Analyzing your week across all modules…</span>
          </div>
          <style>{`@keyframes spin { to { transform: rotate(360deg); } }`}</style>
        </div>
      )}

      {weeklyReview && !loading && (
        <>
          <div style={{ background: "linear-gradient(135deg, rgba(245,158,11,0.06), rgba(245,158,11,0.02))", border: "1px solid rgba(245,158,11,0.2)", borderRadius: "var(--r)", padding: 20, marginBottom: 12 }}>
            <div style={{ display: "flex", alignItems: "center", gap: 10, marginBottom: 14 }}>
              <Ic d={Icons.sparkle} size={18} />
              <div>
                <div style={{ fontWeight: 700, fontSize: 16 }}>Weekly Report</div>
                <div className="caption">Generated {weeklyReview.generatedOn}</div>
              </div>
            </div>
            <div style={{ fontSize: 14, lineHeight: 1.8, color: "var(--muted2)", marginBottom: 18 }}>{weeklyReview.summary}</div>

            {weeklyReview.suggestions?.length > 0 && (
              <>
                <div className="label" style={{ marginBottom: 10 }}>Suggestions</div>
                <div style={{ display: "flex", flexDirection: "column", gap: 8, marginBottom: 16 }}>
                  {weeklyReview.suggestions.map((s, i) => (
                    <div key={i} style={{ background: "var(--s2)", borderRadius: 10, padding: "12px 14px", borderLeft: "3px solid var(--amber)" }}>
                      <div style={{ fontSize: 13, fontWeight: 700, color: "var(--text)", marginBottom: 3 }}>{s.title}</div>
                      <div style={{ fontSize: 13, color: "var(--muted2)" }}>{s.text}</div>
                    </div>
                  ))}
                </div>
              </>
            )}

            {weeklyReview.planAdjustment && (
              <div style={{ background: "var(--s2)", border: "1px solid var(--border)", borderRadius: 10, padding: 14 }}>
                <div className="label" style={{ marginBottom: 6 }}>Next Week Tweak</div>
                <div style={{ fontSize: 13, color: "var(--text)", lineHeight: 1.6, marginBottom: weeklyReview.planAccepted ? 0 : 12 }}>{weeklyReview.planAdjustment}</div>
                {!weeklyReview.planAccepted ? (
                  <div style={{ display: "flex", gap: 8 }}>
                    <button style={{ flex: 1, padding: "9px", borderRadius: 8, fontSize: 12, fontWeight: 700, cursor: "pointer", background: "var(--amber)", border: "none", color: "#000" }} onClick={() => setWeeklyReview({ ...weeklyReview, planAccepted: true })}>Accept</button>
                    <button style={{ flex: 1, padding: "9px", borderRadius: 8, fontSize: 12, fontWeight: 600, cursor: "pointer", background: "transparent", border: "1px solid var(--border)", color: "var(--muted2)" }} onClick={() => setWeeklyReview({ ...weeklyReview, planAdjustment: null })}>Skip</button>
                  </div>
                ) : <div style={{ fontSize: 12, color: "var(--green)" }}>✓ Accepted</div>}
              </div>
            )}
          </div>

          {canGenerate && (
            <button className="btn btn-ghost" style={{ display: "flex", alignItems: "center", gap: 8, justifyContent: "center" }} onClick={generate}>
              <Ic d={Icons.refresh} size={15} /> Regenerate
            </button>
          )}
        </>
      )}

      <div className="divider" />
      <div className="section-hd"><span className="label">Export Data</span></div>
      <button className="btn btn-ghost" style={{ display: "flex", alignItems: "center", gap: 8, justifyContent: "center" }} onClick={exportCSV}>
        <Ic d={Icons.download} size={16} /> Export all data as CSV
      </button>
    </div>
  );
}

// ─── MODALS ───────────────────────────────────────────────────────────────────
function Modal({ modal, setModal, meals, setMeals, workouts, setWorkouts, showToast }) {
  const close = () => setModal(null);
  if (modal === "add_meal") return <AddMealSheet onSave={m => { setMeals(prev => [...prev, m]); showToast("Meal logged! 🍎"); close(); }} onClose={close} />;
  if (modal === "plan_workout") return <PlanWorkoutSheet onSave={w => { setWorkouts(prev => [...prev, w]); showToast("Workout planned!"); close(); }} onClose={close} />;
  return null;
}

function AddMealSheet({ onSave, onClose }) {
  const [name, setName] = useState("");
  const [cal, setCal] = useState("");
  const [type, setType] = useState("Lunch");
  const types = ["Breakfast", "Lunch", "Dinner", "Snack"];
  const now = new Date();
  const time = `${String(now.getHours()).padStart(2, "0")}:${String(now.getMinutes()).padStart(2, "0")}`;
  return (
    <div className="overlay" onClick={onClose}>
      <div className="sheet" onClick={e => e.stopPropagation()}>
        <div className="sheet-handle" />
        <div className="sheet-title">Add Meal</div>
        <div className="field"><label>Meal Name</label><input className="inp" placeholder="e.g. Oatmeal with banana" value={name} onChange={e => setName(e.target.value)} /></div>
        <div className="field"><label>Calories (kcal)</label><input className="inp" type="number" placeholder="450" value={cal} onChange={e => setCal(e.target.value)} /></div>
        <div className="field">
          <label>Meal Type</label>
          <div className="tags">{types.map(t => <span key={t} className={`tag ${type === t ? "on" : ""}`} onClick={() => setType(t)}>{t}</span>)}</div>
        </div>
        <button className="btn" style={{ marginTop: 8 }} onClick={() => {
          if (!name.trim() || !cal) return;
          onSave({ id: Date.now(), date: TODAY(), name: name.trim(), calories: parseInt(cal), mealType: type, time });
        }}>Log Meal</button>
      </div>
    </div>
  );
}

function PlanWorkoutSheet({ onSave, onClose }) {
  const [name, setName] = useState("");
  const [type, setType] = useState("Push");
  const [duration, setDuration] = useState("");
  const types = ["Push", "Pull", "Legs", "Full Body", "Cardio", "Core"];
  return (
    <div className="overlay" onClick={onClose}>
      <div className="sheet" onClick={e => e.stopPropagation()}>
        <div className="sheet-handle" />
        <div className="sheet-title">Plan Workout</div>
        <div className="field"><label>Workout Name</label><input className="inp" placeholder="e.g. Upper Body A" value={name} onChange={e => setName(e.target.value)} /></div>
        <div className="field">
          <label>Type</label>
          <div className="tags">{types.map(t => <span key={t} className={`tag ${type === t ? "on" : ""}`} onClick={() => setType(t)}>{t}</span>)}</div>
        </div>
        <div className="field"><label>Duration (min, optional)</label><input className="inp" type="number" placeholder="45" value={duration} onChange={e => setDuration(e.target.value)} /></div>
        <button className="btn" style={{ marginTop: 8 }} onClick={() => {
          if (!name.trim()) return;
          onSave({ id: Date.now(), date: TODAY(), name: name.trim(), type, duration: parseInt(duration) || 0, completed: false });
        }}>Save Plan</button>
      </div>
    </div>
  );
}

// ─── MAIN APP ─────────────────────────────────────────────────────────────────
export default function App() {
  const [user, setUser] = useStorage("forge_user_v1", null);
  const [runs, setRuns] = useStorage("forge_runs_v1", []);
  const [focusSessions, setFocusSessions] = useStorage("forge_focus_v1", []);
  const [meals, setMeals] = useStorage("forge_meals_v1", []);
  const [workouts, setWorkouts] = useStorage("forge_workouts_v1", []);
  const [weeklyReview, setWeeklyReview] = useStorage("forge_review_v1", null);
  const [tab, setTab] = useState("home");
  const [modal, setModal] = useState(null);
  const [toast, setToast] = useState(null);

  const showToast = useCallback((msg) => {
    setToast(msg);
    setTimeout(() => setToast(null), 2800);
  }, []);

  if (!user) return (
    <>
      <style>{STYLES}</style>
      <div className="app"><AuthScreen onAuth={setUser} /></div>
    </>
  );

  const ctx = { user, setUser, runs, setRuns, focusSessions, setFocusSessions, meals, setMeals, workouts, setWorkouts, weeklyReview, setWeeklyReview, modal, setModal, showToast };

  return (
    <>
      <style>{STYLES}</style>
      <div className="app">
        {toast && <div className="toast">{toast}</div>}
        {tab === "home"     && <HomeScreen {...ctx} />}
        {tab === "run"      && <RunScreen {...ctx} />}
        {tab === "focus"    && <FocusScreen {...ctx} />}
        {tab === "calories" && <CalorieScreen {...ctx} />}
        {tab === "workout"  && <WorkoutScreen {...ctx} />}
        {tab === "review"   && <ReviewScreen {...ctx} />}
        {modal && <Modal {...ctx} />}
        <BottomNav tab={tab} setTab={(t) => { if (t === "profile") { if (confirm("Sign out?")) setUser(null); } else setTab(t); }} />
      </div>
    </>
  );
}
