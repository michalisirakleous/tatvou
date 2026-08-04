# Daily market scan — automated

Runs every weekday on GitHub's servers, posts the brief as an issue, and
emails it to you. Your computer stays off. You do nothing.

## What arrives

An email each weekday evening with a subject line like:

> Market scan 04 Aug — NVDA +7.0% (z=3.9), 47% advancing

The headline is in the subject, so most days you won't need to open it.
Inside: what moved unusually, sector averages, and headlines for the
biggest movers only.

Every move is divided by that instrument's **own** recent volatility.
A raw percentage means little on its own — MARA down 4% is a quiet day,
SPY down 4% is a crisis. A z-score of 3 means the same thing everywhere.

## Setup — about 10 minutes, once

**1. Create a repository**

Go to [github.com/new](https://github.com/new). Name it `market-scan`.
Choose **Private**. Tick "Add a README file". Click Create.

**2. Add the two files**

In the repo, click **Add file → Upload files**, then drag in `scan.py`.
Commit it.

Then click **Add file → Create new file**. In the filename box type exactly:

```
.github/workflows/daily-scan.yml
```

(the slashes create the folders automatically). Paste the contents of
`daily-scan.yml` into the editor and commit.

**3. Let it write to the repo**

Settings → Actions → General → scroll to **Workflow permissions** →
select **Read and write permissions** → Save.

Without this the run fails at the last step.

**4. Turn on notifications**

Click **Watch** (top right of the repo) → **All Activity**. This is what
sends you the email when an issue is created.

**5. Test it**

Actions tab → **Daily market scan** → **Run workflow** → Run workflow.
Wait about two minutes, then check the Issues tab. If a brief is there,
you're done — it will now run by itself every weekday.

## When it runs

22:00 UTC, Monday to Friday — an hour after the US close. That's midnight
in Cyprus.

GitHub's scheduler is best-effort and can run 5–30 minutes late when the
platform is busy. Occasionally a scheduled run is skipped entirely. This
is fine for a daily brief; it is exactly why you would not want anything
time-critical on it.

To change the time, edit the `cron` line. It is in UTC:

```yaml
- cron: "0 22 * * 1-5"     # minute hour day month weekday
```

## Cost

Free. Public repos get unlimited Actions minutes; private repos get 2,000
a month. This uses roughly 2 minutes a day — about 40 a month.

## Files it keeps

Each brief is committed to `archive/YYYY-MM-DD.md`, so you build a
searchable history of what the market did. `archive/latest_scan.csv` has
the full numeric table from the most recent run.

## If it breaks

The Actions tab shows every run with full logs. The usual causes:

- **Fails on the last step** — workflow permissions not set to read/write (step 3).
- **No email** — you are not watching the repo (step 4).
- **`yfinance` errors** — Yahoo occasionally changes its API. Re-run manually;
  if it persists, `yfinance` needs updating.

## What it will not do

It reports what moved. It never says what to do about it, and there is no
broker connection, no order path, and no credentials anywhere in it.

Something unusual happens every single day. Treating each one as
actionable is how a scanner becomes overtrading. Most days the right
response to this email is to read the subject line and delete it.
