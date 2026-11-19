File 4 — type: data_pipeline/04_scrape_affidavits.py

#!/usr/bin/env python3
"""Scrape all MLA affidavit pages from MyNeta Maharashtra2024 (rendered)."""
import json, os, threading, queue, time
from playwright.sync_api import sync_playwright

from pathlib import Path
S = str(Path(__file__).resolve().parent / "outputs")
OUT = str(Path(__file__).resolve().parent / "affidavits")

os.makedirs(OUT, exist_ok=True)

d = json.load(open(f"{S}/realdata.json"))
targets = []
for m in d["mlas"]:
    if m["cid"] not in ("awhad", "jankar"):
        targets.append((str(m["cid"]), m["name"]))
for c in d["consts"]:
    if c.get("cur", {}).get("isbye"):
        for r in c["cur"]["cands"]:
            if r[8]:
                targets.append((str(r[0]), r[1]))
targets = sorted(set(targets))
targets = [(cid, nm) for cid, nm in targets if not os.path.exists(f"{OUT}/{cid}.html")]
print(f"to fetch: {len(targets)}", flush=True)

Q = queue.Queue()
for t in targets:
    Q.put(t)
done, failed = [], []
lock = threading.Lock()

def worker(n):
    with sync_playwright() as p:
        b = p.chromium.launch(args=["--no-sandbox"])
        pg = b.new_page()
        while True:
            try:
                cid, nm = Q.get_nowait()
            except queue.Empty:
                break
            url = f"https://www.myneta.info/Maharashtra2024/candidate.php?candidate_id={cid}"
            ok = False
            for attempt in range(2):
                try:
                    pg.goto(url, timeout=45000, wait_until="domcontentloaded")
                    pg.wait_for_timeout(700)
                    html = pg.content()
                    if len(html) > 4000 and "candidate" in html.lower():
                        open(f"{OUT}/{cid}.html", "w", encoding="utf-8").write(html)
                        ok = True
                    else:
                        time.sleep(1.5)
                except Exception as e:
                    time.sleep(2)
            with lock:
                (done if ok else failed).append(cid)
                if len(done) % 40 == 0:
                    print(f"[w{n}] {len(done)} done, {len(failed)} failed", flush=True)
        b.close()

threads = [threading.Thread(target=worker, args=(i,)) for i in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(f"DONE: {len(done)} fetched, {len(failed)} failed")
if failed:
    print("FAILED CIDS:", failed)
