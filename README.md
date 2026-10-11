"""生成 dist/status.svg：每次运行时从 GitHub API 取最新数据。"""
import os, json, urllib.request
from datetime import datetime, timezone, timedelta

USER = os.environ.get("GH_USER", "csq66666666")
TOKEN = os.environ.get("GITHUB_TOKEN")


def api(path):
    req = urllib.request.Request(
        "https://api.github.com" + path,
        headers={"Accept": "application/vnd.github+json", "User-Agent": "profile-status"},
    )
    if TOKEN:
        req.add_header("Authorization", "Bearer " + TOKEN)
    with urllib.request.urlopen(req, timeout=20) as r:
        return json.load(r)


def ago(iso):
    dt = datetime.fromisoformat(iso.replace("Z", "+00:00"))
    s = int((datetime.now(timezone.utc) - dt).total_seconds())
    if s < 3600:
        return f"{max(s // 60, 1)}m ago"
    if s < 86400:
        return f"{s // 3600}h ago"
    return f"{s // 86400}d ago"


user = api(f"/users/{USER}")
stars, page = 0, 1
while page <= 5:
    repos = api(f"/users/{USER}/repos?per_page=100&type=owner&page={page}")
    stars += sum(r.get("stargazers_count", 0) for r in repos)
    if len(repos) < 100:
        break
    page += 1

try:
    events = api(f"/users/{USER}/events/public?per_page=1")
    last = ago(events[0]["created_at"]) if events else "-"
except Exception:
    last = "-"

now = datetime.now(timezone(timedelta(hours=8))).strftime("%Y-%m-%d %H:%M UTC+8")
metrics = [
    (str(user.get("public_repos", 0)), "REPOSITORIES"),
    (str(stars), "STARS EARNED"),
    (str(user.get("followers", 0)), "FOLLOWERS"),
    (last, "LAST ACTIVITY"),
]

MONO = "'Fira Code', 'SF Mono', Menlo, Consolas, 'Liberation Mono', monospace"
SANS = "-apple-system, 'SF Pro Display', 'Helvetica Neue', 'Segoe UI', sans-serif"

cols = ""
for i, (val, label) in enumerate(metrics):
    cx = 135 + i * 210
    cols += f"""  <text x="{cx}" y="96" text-anchor="middle" font-size="36" font-weight="700" fill="#fff" font-family="{SANS}">{val}</text>
  <text x="{cx}" y="120" text-anchor="middle" font-size="11.5" letter-spacing="2.5" fill="#fff" fill-opacity="0.45" font-family="{MONO}">{label}</text>
"""

ecg = "M30 146"
x = 30
for bx in range(150, 860, 180):
    ecg += f" H{bx} l8 -10 l9 20 l9 -28 l8 18"
ecg += " H870"

svg = f"""<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 170" width="900" height="170" role="img" aria-label="GitHub live status">
  <defs>
    <linearGradient id="bg" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stop-color="#0B1020"/><stop offset="1" stop-color="#141D3B"/></linearGradient>
    <linearGradient id="edge" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stop-color="#fff" stop-opacity="0.7"/><stop offset="0.4" stop-color="#fff" stop-opacity="0.08"/><stop offset="1" stop-color="#fff" stop-opacity="0.4"/></linearGradient>
    <filter id="glow" x="-50%" y="-50%" width="200%" height="200%"><feGaussianBlur stdDeviation="3"/></filter>
    <clipPath id="c"><rect width="900" height="170" rx="24"/></clipPath>
  </defs>
  <g clip-path="url(#c)">
    <rect width="900" height="170" fill="url(#bg)"/>
    <circle cx="450" cy="-30" r="200" fill="#3B82F6" opacity="0.18" filter="url(#glow)"/>
    <circle cx="36" cy="32" r="5" fill="#4ADE80"/>
    <circle cx="36" cy="32" r="5" fill="none" stroke="#4ADE80"><animate attributeName="r" values="5;14" dur="2s" repeatCount="indefinite"/><animate attributeName="opacity" values="0.8;0" dur="2s" repeatCount="indefinite"/></circle>
    <text x="52" y="36" font-size="12" letter-spacing="3" fill="#fff" fill-opacity="0.6" font-family="{MONO}">LIVE STATUS</text>
    <text x="864" y="36" text-anchor="end" font-size="12" fill="#fff" fill-opacity="0.4" font-family="{MONO}">updated {now}</text>
{cols}    <path d="{ecg}" fill="none" stroke="#fff" stroke-opacity="0.07" stroke-width="2"/>
    <path d="{ecg}" pathLength="1000" fill="none" stroke="#58A6FF" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" stroke-dasharray="170 830">
      <animate attributeName="stroke-dashoffset" values="0;-1000" dur="5s" repeatCount="indefinite"/>
    </path>
  </g>
  <rect x="0.5" y="0.5" width="899" height="169" rx="23.5" fill="none" stroke="url(#edge)"/>
</svg>
"""

os.makedirs("dist", exist_ok=True)
with open("dist/status.svg", "w", encoding="utf-8") as fh:
    fh.write(svg)
print("status.svg written:", metrics)
