File 5 — type: data_pipeline/05_parse_affidavits.py

#!/usr/bin/env python3
"""Parse all affidavit pages -> affdetail.json (v2, correct structure)."""
import json, os, re
from bs4 import BeautifulSoup

from pathlib import Path
S = str(Path(__file__).resolve().parent / "outputs")
NUM = re.compile(r"[\d,]+")
INCPAIR = re.compile(r"(20\d{2}\s*-\s*\d{2,4})\s*\**\s*Rs\s*([\d,]+)")

def rs(x):
    if x is None:
        return None
    x = x.replace("\xa0", " ").strip()
    if not x or x.lower() in ("nil", "-", "n/a", "na", "none", "yes", "no"):
        return 0
    m = NUM.findall(x.replace("Rs", ""))
    return int(m[0].replace(",", "")) if m else 0

def cells(tr):
    return [c.get_text(" ", strip=True) for c in tr.find_all(["td", "th"])]

def parse_page(path):
    soup = BeautifulSoup(open(path, encoding="utf-8").read(), "lxml")
    out = {"inc": [], "cases": [], "conv": [], "mov": [], "imm": [], "liab": [], "prev": {}}
    h2 = soup.find("h2")
    out["pageName"] = h2.get_text(" ", strip=True).replace("(Winner)", "").strip() if h2 else None
    # age
    m = re.search(r"Age[:\s]*(\d+)", soup.get_text(" ", strip=True)[:3000])
    if m:
        out["age"] = int(m.group(1))
    # education text (h3 + following nodes)
    for h in soup.find_all("h3"):
        if h.get_text(strip=True).startswith("Educational Details"):
            buf = []
            sib = h.next_sibling
            while sib is not None and len(" ".join(buf)) < 300:
                if isinstance(sib, str):
                    t = sib.strip()
                elif getattr(sib, "name", None) == "h3":
                    break
                else:
                    t = sib.get_text(" ", strip=True)
                if t:
                    buf.append(t)
                sib = sib.next_sibling
            out["eduDetail"] = " · ".join(b for b in buf if b)[:290] or None
            break
    # tables
    for tb in soup.find_all("table"):
        rows = [cells(r) for r in tb.find_all("tr")]
        if not rows:
            continue
        head = " | ".join(rows[0])
        body = " ".join(" ".join(r) for r in rows)
        if "Relation Type" in head and "Financial Year" in head:
            for r in rows[1:]:
                if len(r) >= 4 and r[0] in ("self", "spouse", "huf", "dependent1", "dependent2", "dependent3"):
                    pairs = INCPAIR.findall(r[3])
                    if pairs:
                        out["inc"].append({"r": r[0], "s": [[fy.replace(" ", ""), v] for fy, v in pairs]})
        elif "FIR No." in head and "Serial No" in head:
            hdr = rows[0]
            for r in rows[1:]:
                if len(r) < 3 or "No Cases" in " ".join(r):
                    continue
                d = {}
                for i, h in enumerate(hdr):
                    hl = h.lower()
                    v = r[i] if i < len(r) else ""
                    if "fir" in hl:
                        d["fir"] = v[:80]
                    elif "case no" in hl:
                        d["no"] = v[:60]
                    elif "court" in hl:
                        d["court"] = v[:70]
                    elif "law /" in hl:
                        d["law"] = v[:40]
                    elif "ipc" in hl or "bns" in hl:
                        d["sec"] = v[:120]
                    elif "other details" in hl:
                        d["oth"] = v[:110]
                    elif "charges framed" in hl:
                        d["ch"] = v[:8]
                if d:
                    out["cases"].append(d)
        elif "Punishment Imposed" in head:
            for r in rows[1:]:
                if len(r) >= 7 and "No Cases" not in " ".join(r):
                    out["conv"].append({"no": r[1][:60], "court": r[2][:70] if len(r) > 2 else "",
                                        "sec": r[4][:120] if len(r) > 4 else "",
                                        "punish": r[6][:80] if len(r) > 6 else "",
                                        "date": r[7][:24] if len(r) > 7 else "",
                                        "appeal": r[9][:70] if len(r) > 9 else ""})
        elif "Description" in head and "self" in head and "spouse" in head:
            kind = None
            if "Agricultural Land" in body:
                kind = "imm"
            elif "Grand Total of Liabilities" in body or "Loans from Banks" in body:
                kind = "liab"
            elif "Cash" in body and "Jewellery" in body:
                kind = "mov"
            if not kind:
                continue
            tot = 0
            for r in rows[1:]:
                if len(r) < 3:
                    continue
                first = r[0].strip()
                desc_col = 1 if re.match(r"^([ivx]+|\d+|\()", first) else 0
                d = re.sub(r"^\(+[a-z0-9]+\)\s*", "", r[desc_col]).strip()[:46]
                v = rs(r[-1])
                if re.match(r"(Totals|Gross Total|Grand Total)", d):
                    if v and "Calculated" in d or d.startswith("Totals"):
                        if kind == "liab" and "Govt Dues" in d:
                            out["govtDues"] = v
                        elif v > tot and (kind != "liab" or "Dues" not in d):
                            tot = max(tot, v)
                    continue
                if v and d:
                    out[kind if kind != "imm" else "imm"].append({"d": d, "v": v})
            out[{"mov": "movT", "imm": "immT", "liab": "liabT"}[kind]] = tot
        elif "Other Elections" in head or "Maharashtra 2019" in body[:200]:
            for r in rows[1:]:
                j = " ".join(r)
                m2 = re.search(r"(20\d\d)", j)
                if m2:
                    vals = [rs(x) for x in r if rs(x)]
                    vs = [v for v in vals if v > 100000]
                    if vs:
                        out["prev"][m2.group(1)] = max(vs)
    # headline assets/liab
    for tb in soup.find_all("table"):
        rows = [cells(r) for r in tb.find_all("tr")]
        if rows and rows[0] and rows[0][0].startswith("Assets"):
            out["totAssets"] = rs(rows[0][1]) if len(rows[0]) > 1 else None
            if len(rows) > 1 and len(rows[1]) > 1:
                out["totLiab"] = rs(rows[1][1])
            break
    return out

d = json.load(open(f"{S}/realdata.json"))
byc = {str(m["cid"]): m for m in d["mlas"] if m["cid"] not in ("awhad", "jankar")}
for c in d["consts"]:
    if c.get("cur", {}).get("isbye"):
        for r in c["cur"]["cands"]:
            if r[8]:
                byc.setdefault(str(r[0]), {"name": r[1], "cases": 0, "assets": 0})

detail, bad = {}, []
for f in sorted(os.listdir(f"{S}/affidavits")):
    cid = f[:-5]
    try:
        detail[cid] = parse_page(f"{S}/affidavits/{f}")
    except Exception as e:
        bad.append((cid, repr(e)))

mism = []
for cid, m in byc.items():
    p = detail.get(cid)
    if not p:
        mism.append((cid, m["name"], "NO PAGE"))
        continue
    ncase = len(p["cases"]) + len(p["conv"])
    if ncase != m["cases"]:
        mism.append((cid, m["name"][:28], f"cases {m['cases']} vs pending {len(p['cases'])}+conv {len(p['conv'])}"))
    ta = p.get("totAssets")
    if ta and m["assets"] and abs(ta - m["assets"]) > max(1000, 0.02 * m["assets"]):
        mism.append((cid, m["name"][:28], f"assets {m['assets']} vs {ta}"))

json.dump(detail, open(f"{S}/affdetail.json", "w"), ensure_ascii=False)
print(f"parsed {len(detail)} pages, {len(bad)} exceptions", bad[:5] if bad else "")
print(f"validation mismatches: {len(mism)}")
for x in mism[:25]:
    print("  ", x)
stats = {
    "with_prev2019": sum(1 for p in detail.values() if "2019" in p.get("prev", {})),
    "with_income_series": sum(1 for p in detail.values() if any(i["r"] == "self" and len(i["s"]) >= 3 for i in p.get("inc", []))),
    "convicted_mlas": [cid for cid, p in detail.items() if p.get("conv")],
    "with_govt_dues": sum(1 for p in detail.values() if p.get("govtDues")),
}
print("STATS:", json.dumps(stats))
p = detail.get("1072")
print("Patil inc self:", [i for i in p["inc"] if i["r"] == "self"])
print("Patil movT/immT/liabT:", p.get("movT"), p.get("immT"), p.get("liabT"), "| mov items:", len(p["mov"]), "| imm items:", len(p["imm"]), "| govtDues:", p.get("govtDues"))
print("Patil mov:", p["mov"][:4])
print("Patil edu:", p.get("eduDetail"))
