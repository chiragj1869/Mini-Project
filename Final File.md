File 6 — type: data_pipeline/06_build_portal_data.py

#!/usr/bin/env python3
"""Build the final dataset the Flask app serves: data/portal_data.json.

Assembles:
  * outputs/realdata.json   - 286 joined MLAs (pipeline steps 01-03)
  * outputs/consts_data.json- all constituency rosters (4,144 candidates)
  * outputs/affdetail.json  - parsed affidavit detail (step 05)
  * research/deep.json      - the 12 researched leader profiles

This step also completes the two partial records (Awhad, Jankar) with
news-sourced / last-known figures and injects their roster rows, then
recomputes the final house statistics. This is the data-filling step:
everything the website shows comes out of this file.
"""
import json
from pathlib import Path

HERE = Path(__file__).resolve().parent
S = str(HERE / "outputs")
OUT = HERE.parent / "data" / "portal_data.json"

d = json.load(open(f"{S}/realdata.json"))
mlas, consts, meta = d["mlas"], d["consts"], d["meta"]
PARTIES, EDUS = meta["parties"], meta["edu"]

# ---- add Awhad (Mumbra-Kalwa) & Jankar (Malshiras): the 2 partial records ----
def mrec(cid, name, ac, acno, district, party, cases, edu, age, assets, liab, votes, share, margin,
         rcand, rparty, rvotes, constId, note2019=None, assetsNoteSrc=None):
    return {"cid": cid, "name": name, "ac": ac, "acno": acno, "district": district, "party": party,
            "cases": cases, "serious": False, "edu": edu, "age": age, "assets": assets, "liab": liab,
            "votes": votes, "share": share, "margin": margin, "rcand": rcand, "rparty": rparty,
            "rvotes": rvotes, "bye": False, "constId": constId, "prevMLA": "", "women": False,
            "note2019": note2019, "assetsNoteSrc": assetsNoteSrc}

EXISTING = {m["cid"] for m in mlas}
if "awhad" not in EXISTING:
    mlas.append(mrec("awhad", "Awhad Jitendra Satish", "Mumbra-Kalwa", 149, "Thane",
        "NCP-Sharadchandra Pawar", -1, "Doctorate", -1, 831400000, -1,
        157141, 61.93, 96228, "Najeeb Mulla", "NCP", 60913, 147,
        note2019={"assets": 478936553, "cases": 25,
                  "url": "https://www.myneta.info/Maharashtra2019/candidate.php?candidate_id=9833"},
        assetsNoteSrc=("https://www.latestly.com/india/politics/maharashtra-assembly-elections-2024-"
                       "bjps-mangal-prabhat-lodha-shiv-senas-pratap-sarnaik-assembly-speaker-rahul-"
                       "narwekar-top-rich-candidates-list-6372320.html")))
if "jankar" not in EXISTING:
    mlas.append(mrec("jankar", "Jankar Uttamrao Shivdas", "Malshiras (SC)", 254, "Solapur",
        "NCP-Sharadchandra Pawar", -1, "Graduate", -1, -1, -1,
        121713, 50.12, 13147, "Ram Satpute", "BJP", 108566, 257,
        note2019={"assets": 152682760, "cases": 11,
                  "url": "https://www.myneta.info/Maharashtra2019/candidate.php?candidate_id=10908"}))

# ---- build the missing constituency records (Mumbra-Kalwa 149, Malshiras 254) ----
cd = json.load(open(f"{S}/consts_data.json"))
results = json.load(open(f"{S}/results.json"))
have = {c["acno"] for c in consts}
for acno, cid_ in ((149, "147"), (254, "257")):
    if acno in have:
        continue
    r = next(x for x in results if x["no"] == acno)
    v = cd[cid_]
    cands = [[c["cid"], c["name"], PARTIES.index(c["party"]), c["cases"], EDUS.index(c["edu"]),
              c["age"], c["assets"], c["liab"], 1 if c["won"] else 0] for c in v["cands"]]
    consts.append({"ac": r["ac"], "acno": acno, "district": r["district"], "wcand": r["wcand"],
                   "wparty": r["wparty"], "wvotes": r["wvotes"], "wshare": r["wshare"],
                   "rcand": r["rcand"], "rparty": r["rparty"], "rvotes": r["rvotes"], "rshare": r["rshare"],
                   "margin": r["margin"], "bye": False, "ge": None,
                   "cur": {"id": int(cid_), "isbye": False, "cands": cands}})
consts.sort(key=lambda c: c["acno"])

# inject their winner rows into the constituency rosters
INJ = {
    149: ["awhad", "Awhad Jitendra Satish", PARTIES.index("NCP-Sharadchandra Pawar"), -1, EDUS.index("Doctorate"), -1, 831400000, -1, 1,
          ("Winner's affidavit row is missing from the live MyNeta page; restored from news reports of his 2024 "
           "filing (assets ₹83.14 Cr) and his 2019 affidavit (education, cases). Other candidates per MyNeta.")],
    254: ["jankar", "Jankar Uttamrao Shivdas", PARTIES.index("NCP-Sharadchandra Pawar"), -1, EDUS.index("Graduate"), -1, -1, -1, 1,
          ("Winner's affidavit row is missing from the live MyNeta page; his 2024 figures were not traced in "
           "processed public records — last known declaration (2019): ₹15.27 Cr assets, 11 cases (linked on "
           "his profile). Other candidates per MyNeta.")],
}
for c in consts:
    if c["acno"] in INJ:
        row = INJ[c["acno"]]
        c["cur"]["cands"].append(row[:9])
        c["cur"]["cands"].sort(key=lambda a: 0 if a[8] else 1)
        c["rosterNote"] = row[9]

# dedupe ge/cur rosters
for c in consts:
    if c.get("ge") and c["ge"]["id"] == c["cur"]["id"]:
        c["ge"] = None

# ---- recompute final meta ----
mlas.sort(key=lambda m: m["acno"])
crore = sum(1 for m in mlas if m["assets"] >= 1e7)
withc = sum(1 for m in mlas if m["cases"] > 0)
ser = sum(1 for m in mlas if m["serious"])
grads = sum(1 for m in mlas if m["edu"] in ("Graduate", "Graduate Professional", "Post Graduate", "Doctorate"))
tot = sum(m["assets"] for m in mlas if m["assets"] > 0)
women = sum(1 for m in mlas if m.get("women"))
meta.update({"mlas": len(mlas), "crorepatis": crore, "withCases": withc, "serious": ser,
             "grads": grads, "totalAssets": tot, "women": women})
print(f"FINAL DATA: {len(mlas)} MLAs | crorepatis {crore} | cases {withc} | serious {ser} | women {women} | total Rs {tot/1e7:,.0f} Cr")

# ---- attach the parsed affidavit detail (case-by-case, ITRs, assets) ----
aff = json.load(open(f"{S}/affdetail.json"))
natt = nconv = 0
for m in mlas:
    a = aff.get(str(m["cid"]))
    if not a:
        continue
    m["aff"] = {"c": a["cases"], "v": a["conv"], "i": a["inc"], "m": a["mov"], "im": a["imm"],
                "l": a["liab"], "mt": a.get("movT") or 0, "it": a.get("immT") or 0,
                "lt": a.get("liabT") or 0, "p": a["prev"]}
    natt += 1
    if a["conv"]:
        nconv += 1
print(f"affidavit detail attached: {natt} | MLAs with declared convictions: {nconv}")

# ---- research layer: Patil flagship + 12 leader profiles + pledge tracker ----
PATIL = json.load(open(str(HERE / "research" / "patil_profile.json")))
DEEPJ = json.load(open(str(HERE / "research" / "deep.json")))

OUT.parent.mkdir(parents=True, exist_ok=True)
json.dump({"DATA": {"meta": meta, "mlas": mlas, "consts": consts}, "PATIL": PATIL, "DEEP": DEEPJ},
          open(OUT, "w"), ensure_ascii=False, separators=(",", ":"))
print("wrote", OUT)
