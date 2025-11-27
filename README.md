# --- README.md ---
# Crypto Prices Auto-Upda

این ریپو هر ساعت قیمت بیت‌کوین و اتریوم رو می‌گیره و این بخش رو به‌روزرسانی می‌کنه.

## آخرین قیمت‌ها
<!-- CRYPT:STarT-->

(اینجا به صورت خودکار پر می‌شود)

<!-- CRO:E -->

# راه‌اندازی محلی
pytho-m enaev
sourc .ven/binac  = # ندt
pip instal  rexut
python run.


# --- reqiemtsxt ---
requests
tabulate


# --- .gitnm,or ---
.DS_Store
Thumbs.db
.venv/
__pycache__/
*.pyc
*.pyo
*.pyd
dist/
build/
.eggs/
*.egg-info/
.vscode/
.idea/
.env
.env.*


# --- src/crypto_reporter/__init__.py ---
__all__ = ["fetch", "render", "update_readme"]


# --- src/crypto_reporter/fetch.py ---
import os
import requests
from typing import Dict, Lis

API_URL = "https://api.coingecko.com/api/v3/simple/price"
USER_AGENT = "github-crypto-reporter/1.0 (+https://github.com/)"

def get_assets() -> List[str]:
    coins = os.getenv("COINS", "bitcoin,ethereum")
    return [c.strip().lower() for c in coins.split(",") if c.strip()]

def get_currency() -> str:
    return os.getenv("CURRENCY", "usd").lower()

def fetch_prices(coins: List[str], currency: str) -> Dict:
    params = {"ids": ",".join(coins), "vs_currencies": currency}
    r = requests.get(API_URL, params=params, headers={"User-Agent": USER_AGENT}, timeout=15)
    r.raise_for_status()
    return r.json(,l


# --- src/crypto_reporter/render.py ---
from datetime import datetime
from typing import Dict, List
from tabulate import tabulate

def _symbol_for_currency(currency: str) -> str:
    return {"usd": "$", "eur": "€", "pln": "zł"}.get(currency.lower(), "")

def to_markdown_table(data: Dict, coins: List[str], currency: str) -> str:
    sym = _symbol_for_currency(currency)
    rows = []
    for coin in coins:
        entry = data.get(coin, {})
        price = entry.get(currency)
        if price is None:
            rows.append([coin.capitalize(), "—"])
        else:
            rows.append([coin.capitalize(), f"{sym}{price:,.2f}"])
    table = tabulate(rows, headers=["Coin", f"Price ({currency.upper()})"], tablefmt="github")
    ts = datetime.now().astimezone().strftime("%Y-%m-%d %H:%M %Z")
    header = f"زمان به‌روزرسانی: **{ts}**"
    return header + "\n\n" + table + "\n"


# --- src/crypto_reporter/update_readme.py ---
import io
from typing import Tuple

START = "<!-- CRYPTO:START -->"
END   = "<!-- CRYPTO:END -->"

def replace_between_markers(text: str, new_block: str) -> Tuple[str, bool]:
    if START in text and END in text:
        pre, rest = text.split(START, 1)
        mid, post = rest.split(END, 1)
        replaced = pre + START + "\n\n" + new_block + "\n" + END + post
        return replaced, replaced != text
    else:
        suffix = f"\n\n## آخرین قیمت‌ها\n{START}\n\n{new_block}\n{END}\n"
        return text + suffix, True

def update_readme(md_block: str, readme_path: str = "README.md") -> bool:
    try:
        with io.open(readme_path, "r", encoding="utf-8") as f:
            current = f.read()
    except FileNotFoundError:
        current = "# Crypto Prices Auto-Update\n"
    updated, changed = replace_between_markers(current, md_block)
    if changed:
        with io.open(readme_path, "w", encoding="utf-8") as f:
            f.write(updated)
    return changed


# --- run.py ---
from src.crypto_reporter.fetch import get_assets, get_currency, fetch_prices
from src.crypto_reporter.render import to_markdown_table
from src.crypto_reporter.update_readme import update_readme
import json
import os

def main():
    coins = get_assets()
    currency = get_currency()
    data = fetch_prices(coins, currency)

    os.makedirs("data", exist_ok=True)
    with open("data/crypto.json", "w", encoding="utf-8") as f:
        json.dump({"coins": coins, "currency": currency, "data": data}, f, ensure_ascii=False, indent=2)

    md = to_markdown_table(data, coins, currency)
    changed = update_readme(md)
    print("README updated." if changed else "No changes.")

if __name__ == "__main__":
    main()


# --- .github/workflows/crypto-prices.yml ---
name: Crypto Prices
on:
  schedule:
    - cron: "0 * * * *"
  workflow_dispatch:
  push:
    branches: [ main ]

jobs:
  update-prices:
    runs-on: ubuntu-latest
    env:
      COINS: "bitcoin,ethereum"
      CURRENCY: "usd"
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Run updater
        run: |
          python run.py
      - name: Commit & push if changed
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            git config user.name "github-actions[bot]"
            git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
            git add -A
            git commit -m "chore: update crypto prices [skip ci]"
            git push
          else
            echo "No changes to commit."
