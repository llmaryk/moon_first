# --- README.md ---
# Project Title
این ریپو روزانه وضعیت هوا را به‌روزرسانی می‌کند.

## وضعیت هوا
<!-- WEATHER:START -->

(اینجا به صورت خودکار پر می‌شود)

<!-- WEATHER:END -->

# --- setup.cfg ---
[metadata]
name = weather-reporter
version = 0.1.0
description = Daily weather fetcher for README
license = MIT

[options]
package_dir =
    = src
packages = find:
install_requires =
    requests

[options.packages.find]
where = src

# --- requirements.txt ---
requests

# --- .gitignore ---
.DS_Store
Thumbs.db
.venv/
__pycache__/
*.pyc
*.pyo
*.pyd
.build/
dist/
.eggs/
*.egg-info/
.vscode/
.idea/
.env
.env.*

# --- src/weather_reporter/__init__.py ---
__all__ = ["fetch", "render", "update_readme"]

# --- src/weather_reporter/fetch.py ---
import os
import time
import json
from typing import List, Dict
import requests
from datetime import datetime, timezone

WTTR_URL = "https://wttr.in/{city}?format=j1"
USER_AGENT = "github-weather-reporter/1.0 (+https://github.com/)"

def get_cities() -> List[str]:
    env = os.getenv("CITIES", "Tehran, Warsaw, London, New York")
    return [c.strip() for c in env.split(",") if c.strip()]

def fetch_city(city: str) -> Dict:
    url = WTTR_URL.format(city=city)
    r = requests.get(url, headers={"User-Agent": USER_AGENT}, timeout=20)
    r.raise_for_status()
    data = r.json()
    current = data["current_condition"][0]
    feels_like_c = current["FeelsLikeC"]
    temp_c = current["temp_C"]
    humidity = current["humidity"]
    weather_desc = current["weatherDesc"][0]["value"]
    return {
        "city": city,
        "temp_c": int(temp_c),
        "feels_like_c": int(feels_like_c),
        "humidity": int(humidity),
        "desc": weather_desc,
    }

def fetch_all(cities: List[str]) -> Dict:
    out = {"timestamp_utc": datetime.now(timezone.utc).isoformat(), "items": []}
    for c in cities:
        try:
            out["items"].append(fetch_city(c))
            time.sleep(0.6)
        except Exception as e:
            out["items"].append({"city": c, "error": str(e)})
    return out

def save_json(payload: Dict, path: str = "data/weather.json"):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        json.dump(payload, f, ensure_ascii=False, indent=2)

# --- src/weather_reporter/render.py ---
from datetime import datetime
from typing import Dict

def to_markdown_table(payload: Dict) -> str:
    rows = ["| City | Temp (°C) | Feels Like | Humidity (%) | Status |",
            "|---|---:|---:|---:|---|"]
    for item in payload.get("items", []):
        if "error" in item:
            rows.append(f"| {item['city']} | – | – | – | ❌ {item['error']} |")
        else:
            rows.append(
                f"| {item['city']} | {item['temp_c']} | {item['feels_like_c']} | {item['humidity']} | {item['desc']} |"
            )
    ts = payload.get("timestamp_utc")
    ts_local = datetime.fromisoformat(ts).astimezone().strftime("%Y-%m-%d %H:%M %Z") if ts else "unknown"
    header = f"آخرین به‌روزرسانی: **{ts_local}**"
    return header + "\n\n" + "\n".join(rows)

def save_docs(md: str, path: str = "docs/weather.md"):
    import os
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        f.write(md)

# --- src/weather_reporter/update_readme.py ---
import io
import sys
from typing import Tuple

START = "<!-- WEATHER:START -->"
END   = "<!-- WEATHER:END -->"

def replace_between_markers(text: str, new_block: str) -> Tuple[str, bool]:
    if START in text and END in text:
        pre, rest = text.split(START, 1)
        mid, post = rest.split(END, 1)
        replaced = pre + START + "\n\n" + new_block + "\n\n" + END + post
        return replaced, replaced != text
    else:
        suffix = f"\n\n{START}\n\n{new_block}\n\n{END}\n"
        return text + suffix, True

def update_readme(md_block: str, readme_path: str = "README.md") -> bool:
    try:
        with io.open(readme_path, "r", encoding="utf-8") as f:
            current = f.read()
    except FileNotFoundError:
        current = "# Project\n"
    updated, changed = replace_between_markers(current, md_block)
    if changed:
        with io.open(readme_path, "w", encoding="utf-8") as f:
            f.write(updated)
    return changed

if __name__ == "__main__":
    md = sys.stdin.read()
    changed = update_readme(md)
    print("README updated." if changed else "No changes.")

# --- run.py ---
from weather_reporter.fetch import get_cities, fetch_all, save_json
from weather_reporter.render import to_markdown_table, save_docs
import subprocess

def main():
    cities = get_cities()
    payload = fetch_all(cities)
    save_json(payload)
    md = to_markdown_table(payload)
    save_docs(md)
    p = subprocess.run(
        ["python", "-m", "weather_reporter.update_readme"],
        input=md.encode("utf-8"),
        capture_output=True
    )
    print(p.stdout.decode() or p.stderr.decode())

if __name__ == "__main__":
    main()

# --- .github/workflows/weather-report.yml ---
name: Weather Report
on:
  schedule:
    - cron: "0 5 * * *"
  workflow_dispatch:
  push:
    branches: [ main ]
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      CITIES: "Tehran, Warsaw, London, New York"
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -e .
      - name: Run script
        run: |
          python run.py
      - name: Commit & Push if changed
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            git config user.name "github-actions[bot]"
            git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
            git add -A
            git commit -m "chore: update weather report [skip ci]"
            git push
          else
            echo "No changes to commit."

# --- docs/weather.md ---
# (در اولین اجرا تولید می‌شود)

# --- data/weather.json ---
{}

# --- tests/test_render.py ---
from weather_reporter.render import to_markdown_table
def test_table_renders():
    payload = {
        "timestamp_utc": "2025-08-13T05:00:00+00:00",
        "items": [{"city":"Tehran","temp_c":30,"feels_like_c":32,"humidity":40,"desc":"Sunny"}]
    }
    md = to_markdown_table(payload)
    assert "Tehran" in md and "Sunny" in md and "|" in md
