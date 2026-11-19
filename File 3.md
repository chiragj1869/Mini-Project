File 3 — type: data_pipeline/03_patch_records.py



#!/usr/bin/env python3
"""Patch the joined dataset: record corrections, women flags, bypoll details.

Run AFTER 02_join_dataset.py. This step applies the manual verification work:
  1. Mahale (Chikhali) asset correction   - recovered from her affidavit page
     (her row was one of the 'packed' rows MyNeta renders obfuscated).
  2. Women-MLA flags                      - cross-checked against ADR's list.
  3. Baramati & Rahuri 2026 bye-election details (votes, margins, notes).
  4. Recompute the house statistics.

NOTE ON AWHAD & JANKAR (Mumbra-Kalwa, Malshiras): their winner rows are
missing from MyNeta's live pages AND from December-2024 web archives, and
ADR's winners report analysed only 286 of 288 affidavits - the same two.
Every recovery route was tried and documented (see research/ notes).
Their records are therefore completed in 06_build_portal_data.py from
news reports of Awhad's 2024 filing and his 2019 affidavit, and Jankar's
2019 affidavit - each linked on his profile. No source, no number.
"""
import json
from pathlib import Path

S = str(Path(__file__).resolve().parent / "outputs")
d = json.load(open(f"{S}/realdata.json"))
mlas, consts = d["mlas"], d["consts"]
meta = d["meta"]

# --- 1. Mahale asset fix (from her MyNeta candidate page, cid 198) ---
for m in mlas:
    if m["cid"] == "198":
        m["assets"], m["liab"] = 71740028, 22153590
        print("fixed Mahale:", m["assets"])

# --- 2. women flags (cross-checked against ADR women-winners list) ---
women_cids = {r["cid"] for r in json.load(open(f"{S}/list_winner_women.json"))}
for m in mlas:
    m["women"] = m["cid"] in women_cids
print("women MLAs flagged:", sum(1 for m in mlas if m["women"]))

# --- 3. bypoll details for Baramati & Rahuri ---
BYE = {
    201: {"byeVotes": 218969, "byeMargin": 218034, "byeShare": 97.81,
          "note": ("Bye-election 2026 (result 4 May 2026): Sunetra Ajit Pawar (NCP) won with a record margin; "
                   "all major parties had withdrawn candidates. She is Deputy Chief Minister of Maharashtra.")},
    220: {"byeVotes": 140093, "byeMargin": 112587, "byeShare": None,
          "note": ("Bye-election 2026 (result 4 May 2026): Akshay Shivajirao Kardile (BJP) defeated Govind Mokate "
                   "(NCP-SP) after the seat was vacated by the demise of his father, sitting MLA Shivaji Bhanudas Kardile.")},
}
for m in mlas:
    if m["acno"] in BYE:
        m.update(BYE[m["acno"]])
for c in consts:
    if c["acno"] in BYE:
        c["byeDetail"] = BYE[c["acno"]]

# --- 4. sort + recompute meta ---
mlas.sort(key=lambda m: m["acno"])
crore = sum(1 for m in mlas if m["assets"] >= 10**7)
withc = sum(1 for m in mlas if m["cases"] > 0)
ser = sum(1 for m in mlas if m["serious"])
grads = sum(1 for m in mlas if m["edu"] in ("Graduate", "Graduate Professional", "Post Graduate", "Doctorate"))
tot = sum(m["assets"] for m in mlas)
women = sum(1 for m in mlas if m["women"])
meta.update({"mlas": len(mlas), "crorepatis": crore, "withCases": withc, "serious": ser,
             "grads": grads, "totalAssets": tot, "women": women,
             "candidates": sum(len(c["cur"]["cands"]) + (len(c["ge"]["cands"]) if c["ge"] else 0) for c in consts)})
print(f"\nFINAL: MLAs {len(mlas)} | crorepatis {crore} | cases {withc} | serious {ser} | grad+ {grads} | women {women}")
print(f"total declared assets: Rs {tot:,}")
print("ADR reference (Dec-2024 GE winners): 279 crorepatis, 188 cases, 116 serious, 168 grads, 23 women")
json.dump(d, open(f"{S}/realdata.json", "w"), ensure_ascii=False)
print("wrote realdata.json")
