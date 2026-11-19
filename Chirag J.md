# Mini-Project - CHIRAG JADHAV'S WORK

FILE I - data_pipeline/01_scrape_winners_and_constituencies.py

#!/usr/bin/env python3
"""Scrape MyNeta Maharashtra 2024 (rendered via Playwright so packed rows appear)."""
import json, re, sys, os
from playwright.sync_api import sync_playwright
from bs4 import BeautifulSoup

BASE = "https://myneta.info/Maharashtra2024/"
from pathlib import Path
OUT = str(Path(__file__).resolve().parent / "outputs")
BLOCK = ("googletagmanager.com", "google-analytics.com", "doubleclick.net",
         "gstatic.com", "adservice", "amazon-adsystem", "scorecardresearch")

def num(s):
    m = re.search(r'([\d,]+)', s.replace('\xa0', ' '))
    return int(m.group(1).replace(',', '')) if m else 0

def get_soup(page, url, tries=2):
    for t in range(tries):
        try:
            page.goto(url, wait_until="domcontentloaded", timeout=45000)
            page.wait_for_timeout(600)
            return BeautifulSoup(page.content(), 'html.parser')
        except Exception as e:
            if t == tries - 1:
                print("FAIL", url, str(e)[:80])
                return None
            page.wait_for_timeout(1500)

def parse_winners(soup):
    recs = []
    for tr in soup.find_all('tr'):
        tds = tr.find_all('td')
        if len(tds) < 8:
            continue
        if not tds[0].get_text(strip=True).isdigit():
            continue
        a = tr.find('a', href=re.compile('candidate_id='))
        recs.append({
            'sno': int(tds[0].get_text(strip=True)),
            'cid': re.search(r'candidate_id=(\d+)', a['href']).group(1) if a else '0',
            'name': tds[1].get_text(' ', strip=True),
            'ac': tds[2].get_text(strip=True),
            'party': tds[3].get_text(strip=True),
            'cases': num(tds[4].get_text()),
            'edu': tds[5].get_text(strip=True),
            'assets': num(tds[6].get_text()),
            'liab': num(tds[7].get_text()),
        })
    return recs

def parse_const(soup, title):
    m = re.search(r'List of Candidates in (.+?)\s*:\s*(.+?)\s*Maharashtra', title or '')
    if not m:
        return None
    ac, dist = m.group(1).strip(), m.group(2).strip()
    cands = []
    for tr in soup.find_all('tr'):
        tds = tr.find_all('td')
        if len(tds) < 8:
            continue
        if not tds[0].get_text(strip=True).isdigit():
            continue
        a = tr.find('a', href=re.compile('candidate_id='))
        rowtext = tr.get_text(' ', strip=True)
        name = re.sub(r'\s*Winner\s*$', '', tds[1].get_text(' ', strip=True)).strip()
        cands.append({
            'cid': re.search(r'candidate_id=(\d+)', a['href']).group(1) if a else '0',
            'name': name,
            'party': tds[2].get_text(strip=True),
            'cases': num(tds[3].get_text()),
            'edu': tds[4].get_text(strip=True),
            'age': num(tds[5].get_text()),
            'assets': num(tds[6].get_text()),
            'liab': num(tds[7].get_text()),
            'won': 'Winner' in rowtext,
        })
    return {'ac': ac, 'district': dist, 'cands': cands}

def main():
    mode = sys.argv[1]
    with sync_playwright() as p:
        browser = p.chromium.launch(args=["--no-sandbox"])
        ctx = browser.new_context(user_agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36")
        def rt(route):
            if any(d in route.request.url for d in BLOCK):
                route.abort()
            else:
                route.continue_()
        ctx.route("**/*", rt)
        page = ctx.new_page()

        if mode == "winners":
            sort = sys.argv[2] if len(sys.argv) > 2 else "default"
            soup = get_soup(page, BASE + f"index.php?action=show_winners&sort={sort}")
            recs = parse_winners(soup)
            json.dump(recs, open(f"{OUT}/winners_{sort}.json", "w"), ensure_ascii=False)
            crores = sum(1 for r in recs if r['assets'] >= 10**7)
            cases = sum(1 for r in recs if r['cases'] > 0)
            grads = sum(1 for r in recs if r['edu'] in ('Graduate', 'Graduate Professional', 'Post Graduate', 'Doctorate'))
            print(f"WINNERS {sort}: {len(recs)} | crorepatis {crores} (ADR 279) | cases {cases} (ADR 188) | grad+ {grads} (ADR 168)")

        elif mode == "serious":
            ids = set()
            for pg in range(1, 9):
                soup = get_soup(page, f"{BASE}index.php?action=summary&subAction=winner_serious_crime&sort=party&page={pg}")
                if soup is None:
                    break
                txt = soup.get_text()
                found = set(re.findall(r'candidate_id=(\d+)', str(soup)))
                if not found:
                    print(f"page {pg}: no more rows, stopping")
                    break
                ids |= found
                print(f"page {pg}: +{len(found)}")
            old = set(open(f"{OUT}/serious_ids.txt").read().split()) if os.path.exists(f"{OUT}/serious_ids.txt") else set()
            allids = ids | old
            open(f"{OUT}/serious_ids.txt", "w").write("\n".join(sorted(allids)))
            print("TOTAL SERIOUS IDS:", len(allids))

        elif mode == "lists":
            # summary category lists: rows same shape as winners table
            import sys as _s
            sub = _s.argv[2]
            allrows = []
            for pg in range(1, 25):
                soup = get_soup(page, f"{BASE}index.php?action=summary&subAction={sub}&sort=party&page={pg}")
                if soup is None:
                    break
                rows = parse_winners(soup)
                if not rows:
                    print(f"{sub} page {pg}: end")
                    break
                allrows.extend(rows)
                print(f"{sub} page {pg}: +{len(rows)}")
            json.dump(allrows, open(f"{OUT}/list_{sub}.json", "w"), ensure_ascii=False)
            print(f"{sub} TOTAL: {len(allrows)} rows, {len(set(r['cid'] for r in allrows))} unique cids")

        elif mode == "const":
            start, end = int(sys.argv[2]), int(sys.argv[3])
            data = {}
            path = f"{OUT}/consts_data.json"
            if os.path.exists(path):
                data = json.load(open(path))
            for cid in range(start, end + 1):
                key = str(cid)
                if key in data:
                    continue
                soup = get_soup(page, f"{BASE}index.php?action=show_candidates&constituency_id={cid}")
                if soup is None:
                    continue
                title = soup.title.get_text() if soup.title else ""
                if 'Page Not Found' in title:
                    continue
                parsed = parse_const(soup, title)
                if not parsed or not parsed['cands']:
                    print(f"const {cid}: UNPARSEABLE title={title[:70]!r}")
                    continue
                data[key] = parsed
                if len(data) % 25 == 0:
                    json.dump(data, open(path, "w"), ensure_ascii=False)
                    print(f"...saved {len(data)} ACs (last id {cid}: {parsed['ac']}, {len(parsed['cands'])} cands)", flush=True)
            json.dump(data, open(path, "w"), ensure_ascii=False)
            acs = set(v['ac'].upper() for v in data.values())
            tot = sum(len(v['cands']) for v in data.values())
            wins = sum(1 for v in data.values() for c in v['cands'] if c.get('won'))
            print(f"CONSTS: {len(data)} pages | distinct ACs {len(acs)} | candidates {tot} | winners marked {wins}")
        browser.close()

if __name__ == "__main__":
    main()
