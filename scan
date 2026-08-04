"""
Daily market scan, written for unattended runs.

Same logic as the notebook version, but it writes a markdown brief to
disk instead of printing tables, so GitHub Actions can post it as an
issue and email it to you.

Every move is divided by that instrument's OWN recent volatility. A raw
percentage is close to meaningless on its own -- MARA falling 4% is a
quiet day, SPY falling 4% is a crisis. A z-score of 3 means the same
thing everywhere.

NO TRADING. It reports what moved. It never says what to do about it.
"""

import html
import math
import os
import re
import sys
import time
import urllib.parse
import urllib.request
import xml.etree.ElementTree as ET
from datetime import datetime, timedelta, timezone

import numpy as np
import pandas as pd

UNIVERSE = {
    "Index":      ["SPY", "QQQ", "IWM", "DIA", "EFA", "EEM"],
    "Bonds/FX":   ["TLT", "IEF", "HYG", "LQD", "UUP", "FXE"],
    "Commodity":  ["GLD", "SLV", "USO", "DBC", "URA"],
    "Mega tech":  ["AAPL", "MSFT", "GOOGL", "AMZN", "META", "NVDA", "TSLA"],
    "Semis":      ["AMD", "INTC", "MU", "AVGO", "TSM", "SMH"],
    "Financials": ["JPM", "BAC", "GS", "XLF", "V", "MA"],
    "Health":     ["JNJ", "PFE", "UNH", "LLY", "XLV"],
    "Energy":     ["XOM", "CVX", "XLE", "SLB"],
    "Consumer":   ["WMT", "COST", "MCD", "NKE", "XLY", "XLP"],
    "Volatile":   ["COIN", "PLTR", "MARA", "GME", "RIVN", "SOFI"],
}

VOL_WINDOW = 60
Z_THRESHOLD = 2.0
NEWS_FOR_TOP = 6
UA = "Mozilla/5.0 (compatible; market-scan/1.0)"


def scan(universe, vol_window=VOL_WINDOW):
    import yfinance as yf

    tickers = [t for g in universe.values() for t in g]
    sector_of = {t: s for s, g in universe.items() for t in g}
    data = yf.download(tickers, period="6mo", progress=False,
                       auto_adjust=True, group_by="ticker", threads=True)

    rows = []
    for t in tickers:
        try:
            df = data[t] if isinstance(data.columns, pd.MultiIndex) else data
            c = df["Close"].dropna().astype(float)
            v = df["Volume"].dropna().astype(float)
            if len(c) < vol_window + 5:
                continue

            ret = c.pct_change().dropna()
            today = float(ret.iloc[-1])
            base = ret.iloc[-(vol_window + 1):-1]   # excludes today
            sd = float(base.std())
            z = today / sd if sd > 0 else float("nan")

            vr = (float(v.iloc[-1] / v.iloc[-21:-1].mean())
                  if len(v) > 21 and v.iloc[-21:-1].mean() > 0 else float("nan"))

            d = c.diff()
            up = d.clip(lower=0).ewm(alpha=1 / 14, adjust=False).mean()
            dn = (-d).clip(lower=0).ewm(alpha=1 / 14, adjust=False).mean()
            rsi = float((100 - 100 / (1 + up / dn.replace(0, np.nan))).iloc[-1])

            rows.append({
                "ticker": t, "sector": sector_of[t],
                "last": round(float(c.iloc[-1]), 2),
                "chg_%": round(today * 100, 2),
                "z": round(z, 2) if not math.isnan(z) else None,
                "vol_x": round(vr, 2) if not math.isnan(vr) else None,
                "wk_%": round(float(c.pct_change(5).iloc[-1]) * 100, 2),
                "vs_hi_%": round(float((c.iloc[-1] / c.max() - 1) * 100), 1),
                "rsi": round(rsi, 1),
            })
        except Exception as exc:
            print("  ! %s: %s" % (t, str(exc)[:60]), file=sys.stderr)

    return pd.DataFrame(rows)


def news_for(ticker, limit=3):
    url = ("https://feeds.finance.yahoo.com/rss/2.0/headline?s=%s&region=US&lang=en-US"
           % urllib.parse.quote(ticker))
    try:
        req = urllib.request.Request(url, headers={"User-Agent": UA})
        with urllib.request.urlopen(req, timeout=15) as r:
            root = ET.fromstring(r.read())
    except Exception:
        return []

    cutoff = datetime.now(timezone.utc) - timedelta(days=3)
    out = []
    for node in root.iter("item"):
        title = (node.findtext("title") or "").strip()
        if not title:
            continue
        pub = None
        for f in ("%a, %d %b %Y %H:%M:%S %z", "%a, %d %b %Y %H:%M:%S %Z"):
            try:
                pub = datetime.strptime((node.findtext("pubDate") or "").strip(), f)
                pub = pub if pub.tzinfo else pub.replace(tzinfo=timezone.utc)
                break
            except (ValueError, TypeError):
                continue
        if pub and pub < cutoff:
            continue
        out.append({"title": html.unescape(title),
                    "link": (node.findtext("link") or "").strip()})
        if len(out) >= limit:
            break
    return out


def read_the_room(df):
    lines = []
    adv = float((df["chg_%"] > 0).mean() * 100)
    if adv >= 80:
        lines.append("**Broad rally** — %.0f%% of the universe advanced. With breadth "
                     "this wide, individual stock news matters less than the market move." % adv)
    elif adv <= 20:
        lines.append("**Broad selloff** — only %.0f%% advanced. Names down today are "
                     "mostly moving with the market, not on their own news." % adv)
    else:
        lines.append("**Mixed tape** — %.0f%% advanced. Moves are more likely to be "
                     "stock-specific than market-wide." % adv)

    z = df["z"].dropna().abs()
    n = int((z >= Z_THRESHOLD).sum())
    if n == 0:
        lines.append("No move exceeded %.1f standard deviations. A quiet day; nothing "
                     "here demands attention." % Z_THRESHOLD)
    else:
        lines.append("%d name%s moved more than %.1f standard deviations from its own "
                     "normal." % (n, "" if n == 1 else "s", Z_THRESHOLD))

    sec = df.groupby("sector")["chg_%"].mean().sort_values()
    lines.append("Weakest sector: **%s** (%.2f%%). Strongest: **%s** (%.2f%%)."
                 % (sec.index[0], sec.iloc[0], sec.index[-1], sec.iloc[-1]))
    return lines


def build_brief(df):
    today = datetime.now(timezone.utc).strftime("%A %d %B %Y")
    L = ["# Market scan — %s" % today, "",
         "*Information only. No recommendations. Not investment advice.*", ""]

    L += ["## The day", ""] + ["- " + x for x in read_the_room(df)] + [""]

    df = df.copy()
    df["abs_z"] = df["z"].abs()
    ordered = df.dropna(subset=["z"]).sort_values("abs_z", ascending=False)
    unusual = ordered[ordered["abs_z"] >= Z_THRESHOLD]
    show = unusual if len(unusual) else ordered.head(5)

    L += ["## %s" % ("Unusual moves" if len(unusual)
                     else "Nothing crossed the threshold — closest were"), ""]
    cols = ["ticker", "sector", "last", "chg_%", "z", "vol_x", "wk_%", "vs_hi_%", "rsi"]
    L += [show[cols].to_markdown(index=False), "",
          "`z` = today's move in standard deviations of that name's own normal. "
          "`vol_x` = volume vs its 20-day average.", ""]

    sec = df.groupby("sector").agg(names=("ticker", "count"),
                                   avg_chg=("chg_%", "mean"),
                                   max_abs_z=("abs_z", "max")).round(2)
    L += ["## Sectors", "", sec.sort_values("avg_chg").to_markdown(), ""]

    top = show.head(NEWS_FOR_TOP)
    if len(top):
        L += ["## Why? — headlines for the biggest movers", ""]
        for _, r in top.iterrows():
            L.append("**%s** (%+.2f%%, z=%.1f)" % (r["ticker"], r["chg_%"], r["z"]))
            items = news_for(r["ticker"])
            if not items:
                L.append("- no recent headlines found")
            for it in items:
                L.append("- [%s](%s)" % (it["title"], it["link"]))
            L.append("")
            time.sleep(0.4)

    L += ["---", "",
          "*Something unusual happens every day. Treating each one as actionable "
          "is how a scanner turns into overtrading. Most days the right response "
          "to this is to read it and do nothing.*"]
    return "\n".join(L)



def build_html(df):
    """Self-contained dashboard page. No frameworks, no external requests."""
    now = datetime.now(timezone.utc)
    d = df.copy()
    d["abs_z"] = d["z"].abs()
    ordered = d.dropna(subset=["z"]).sort_values("abs_z", ascending=False)
    unusual = ordered[ordered["abs_z"] >= Z_THRESHOLD]
    show = unusual if len(unusual) else ordered.head(6)

    adv = float((df["chg_%"] > 0).mean() * 100)
    sec = df.groupby("sector")["chg_%"].mean().sort_values()

    def row(r):
        col = "up" if r["chg_%"] > 0 else "dn"
        news = news_for(r["ticker"], 2)
        links = "".join(
            '<a href="%s" target="_blank" rel="noreferrer">%s</a>'
            % (i["link"], html.escape(i["title"][:95])) for i in news)
        return ("""<div class="card">
          <div class="hd">
            <span class="tk">%s</span><span class="sec">%s</span>
            <span class="chg %s">%+.2f%%</span>
          </div>
          <div class="mt">
            <span><b>z %+.1f</b> vs its own normal</span>
            <span>volume %sx</span><span>RSI %.0f</span>
            <span>%.1f%% off 52w high</span>
          </div>
          <div class="news">%s</div>
        </div>""" % (r["ticker"], r["sector"], col, r["chg_%"], r["z"],
                     r["vol_x"] if r["vol_x"] else "-", r["rsi"], abs(r["vs_hi_%"]),
                     links or '<span class="none">no recent headlines</span>'))

    tone = ("Broad rally" if adv >= 80 else "Broad selloff" if adv <= 20 else "Mixed tape")
    note = ("individual stock news matters less than the market move" if adv >= 80
            else "names down today are mostly moving with the market" if adv <= 20
            else "moves are more likely stock-specific than market-wide")

    bars = "".join(
        '<div class="sbar"><span>%s</span>'
        '<div class="track"><div class="fill %s" style="width:%.0f%%"></div></div>'
        '<b class="%s">%+.2f%%</b></div>'
        % (name, "up" if val > 0 else "dn",
           min(abs(val) / max(sec.abs().max(), 0.01) * 100, 100),
           "up" if val > 0 else "dn", val)
        for name, val in sec.items())

    return """<!DOCTYPE html>
<html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<meta http-equiv="refresh" content="1800">
<title>Market scan</title><style>
:root{--bg:#EDF0F3;--card:#fff;--ink:#141B26;--mut:#5C6878;--rule:#D2D9E0;
--up:#2F6B4F;--dn:#A63D40;--ind:#2E3A87}
*{box-sizing:border-box}body{margin:0;background:var(--bg);color:var(--ink);
font:15px/1.55 -apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif}
.wrap{max-width:880px;margin:0 auto;padding:26px 18px 60px}
.eyebrow{font-family:ui-monospace,Menlo,Consolas,monospace;font-size:11px;
letter-spacing:.18em;color:var(--mut)}
h1{font-size:26px;margin:8px 0 4px;letter-spacing:-.02em}
.stamp{font-size:12px;color:var(--mut);margin-bottom:22px}
h2{font-size:12px;text-transform:uppercase;letter-spacing:.09em;color:var(--mut);
margin:28px 0 12px}
.summary{background:var(--card);border:1px solid var(--rule);border-left:4px solid var(--ind);
border-radius:4px;padding:18px 20px}
.summary b{font-size:17px}
.card{background:var(--card);border:1px solid var(--rule);border-radius:4px;
padding:15px 18px;margin-bottom:10px}
.hd{display:flex;align-items:baseline;gap:10px;flex-wrap:wrap}
.tk{font-family:ui-monospace,Menlo,monospace;font-weight:700;font-size:17px}
.sec{font-size:12px;color:var(--mut)}
.chg{margin-left:auto;font-family:ui-monospace,Menlo,monospace;font-weight:700;font-size:17px}
.up{color:var(--up)}.dn{color:var(--dn)}
.mt{display:flex;gap:16px;flex-wrap:wrap;font-size:12px;color:var(--mut);margin-top:7px;
font-family:ui-monospace,Menlo,monospace}
.news{margin-top:10px;padding-top:9px;border-top:1px solid var(--rule)}
.news a{display:block;font-size:13px;color:var(--ind);text-decoration:none;padding:2px 0}
.news a:hover{text-decoration:underline}
.none{font-size:12px;color:var(--mut)}
.sbar{display:flex;align-items:center;gap:10px;font-size:13px;margin-bottom:6px}
.sbar span{width:88px;color:var(--mut);font-size:12px}
.sbar b{width:62px;text-align:right;font-family:ui-monospace,Menlo,monospace;font-size:12px}
.track{flex:1;height:16px;background:#E2E7ED;border-radius:2px;overflow:hidden}
.fill{height:100%%}.fill.up{background:var(--up)}.fill.dn{background:var(--dn)}
footer{margin-top:34px;padding-top:16px;border-top:1px solid var(--rule);
font-size:12px;color:var(--mut);line-height:1.6}
</style></head><body><div class="wrap">
<div class="eyebrow">MARKET SCAN &middot; UPDATES ITSELF EACH WEEKDAY</div>
<h1>%s</h1>
<div class="stamp">Last run %s UTC &middot; %d names scanned</div>

<div class="summary">
  <b>%s</b> &mdash; %.0f%% of the universe advanced, so %s.
  %s
</div>

<h2>%s</h2>
%s

<h2>Sectors</h2>
%s

<footer>
Every move is divided by that name's <b>own</b> recent volatility, so a
z-score of 3 means the same thing everywhere. A raw percentage does not.<br><br>
This page reports what moved. It never says what to do about it, and there is
no broker connection or order path anywhere in it. Something unusual happens
every single day &mdash; treating each one as actionable is how a scanner turns
into overtrading. Most days the right response is to read this and do nothing.
</footer>
</div></body></html>""" % (
        now.strftime("%A %d %B %Y"),
        now.strftime("%H:%M"),
        len(df),
        tone, adv, note,
        ("%d name%s moved more than %.1f standard deviations from normal."
         % (len(unusual), "" if len(unusual) == 1 else "s", Z_THRESHOLD))
        if len(unusual) else
        "No move exceeded %.1f standard deviations. A quiet day." % Z_THRESHOLD,
        "Unusual moves" if len(unusual) else "Closest to unusual",
        "".join(row(r) for _, r in show.iterrows()),
        bars)


def main():
    df = scan(UNIVERSE)
    if df.empty:
        print("No data retrieved.", file=sys.stderr)
        return 1

    brief = build_brief(df)
    with open("brief.md", "w", encoding="utf-8") as fh:
        fh.write(brief)

    os.makedirs("docs", exist_ok=True)
    with open("docs/index.html", "w", encoding="utf-8") as fh:
        fh.write(build_html(df))
    df.to_csv("latest_scan.csv", index=False)

    # Headline for the issue title, so the email subject is useful on its own.
    d = df.copy()
    d["abs_z"] = d["z"].abs()
    top = d.dropna(subset=["z"]).sort_values("abs_z", ascending=False).head(1)
    adv = float((df["chg_%"] > 0).mean() * 100)
    if len(top):
        r = top.iloc[0]
        title = ("Market scan %s — %s %+.1f%% (z=%.1f), %.0f%% advancing"
                 % (datetime.now(timezone.utc).strftime("%d %b"),
                    r["ticker"], r["chg_%"], r["z"], adv))
    else:
        title = "Market scan %s" % datetime.now(timezone.utc).strftime("%d %b")

    with open("title.txt", "w", encoding="utf-8") as fh:
        fh.write(title)

    print(title)
    print("Wrote brief.md and docs/index.html")
    return 0


if __name__ == "__main__":
    sys.exit(main())
