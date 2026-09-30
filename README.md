<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=slice&height=180&color=0:04130B,45:16A34A,100:06B6D4&text=Automatic%20Weight%20Recording&fontColor=ffffff&fontSize=32&fontAlign=69&fontAlignY=26&rotate=11&desc=Wyze%20Scale%20%E2%86%92%20n8n%20%E2%86%92%20GitHub%20logging&descAlign=60&descAlignY=45&descSize=18" />
</p>

> **"You can't manage what you don't measure — so I automated the measuring."**

An n8n workflow that reads the latest weight measurement from a Wyze Scale every morning and appends it to [`weight-tracking.md`](https://github.com/Godimas101/personal-projects/blob/main/health-tracking/weight-tracking.md) on GitHub. Step on the scale, walk away, and it's already logged. Very lazy. Very effective.

The key design challenge: Wyze's login endpoint blocks datacenter IPs (first hit on a DigitalOcean droplet), so the server can never call `/api/user/login`. The fix is a **self-refreshing token chain**:
- You log in once from a residential IP and store both tokens on the server.
- On every run, `get_wyze_data.py` calls the refresh endpoint, which is *not* IP-blocked, and writes the new tokens back.

As long as it runs at least once every 28 days, the chain never breaks.

---

## How It Works 🔄

```
Schedule Trigger (daily, 10:00)
  └─ Execute Wyze Script (SSH → get_wyze_data.py on the server)
       └─ Parse Python Output (is the newest reading from today, and not logged yet?)
            └─ New Data Check (passthrough)
                 └─ If shouldLog
                      ├─ true  → Get File → Append New Info → Edit a file → Mark As Logged
                      └─ false → stop, nothing to log
```

1. **Schedule Trigger.** Runs daily at 10:00 in the n8n instance's timezone (`GENERIC_TIMEZONE`; `America/Toronto` here).
2. **Execute Wyze Script.** SSHes into the server and runs `get_wyze_data.py` in its virtualenv. The script refreshes the Wyze tokens, fetches the last 30 days of scale records, and prints the newest one as JSON.
3. **Parse Python Output.**
   - Fails the run if the script reported an error or returned no weight.
   - Otherwise checks two things: whether the reading is from **today in `America/Toronto`**, and whether its `measure_ts` is different from the last reading logged.
   - If either check fails, it returns `shouldLog: false`.
   - If both pass, it formats the fields.
4. **New Data Check.** A passthrough; the checks above already happened.
5. **If.** Continues only when `shouldLog` is `true`.
6. **Get File.** Fetches the tracking file and its SHA from the GitHub REST API. It authenticates with `GITHUB_PAT` from n8n's environment, and retries on failure.
7. **Append New Info.** Decodes the file, appends one table row, re-encodes it, and builds the commit message `chore: auto-log weight YYYY-MM-DD — NNN lbs`.
8. **Edit a file.** Commits the updated file through n8n's GitHub node.
9. **Mark As Logged.** Saves the reading's `measure_ts` in the workflow's static data, so the same reading is never logged twice.

---

## Files 📂

| File | Purpose |
|------|---------|
| `get_wyze_data.py` | **Runs on the server.** Refreshes the tokens, fetches the newest scale record via `wyze-sdk`, and prints it as JSON. |
| `bootstrap-wyze-tokens.py` | **Runs locally.** Break-glass script: logs in from your residential IP and puts fresh tokens on the server. Only needed if the token chain breaks. |
| `credentials.py` | Your local credentials. It's gitignored, so it's never committed. You create it from `credentials.example.py`. |
| `credentials.example.py` | Template showing the fields `credentials.py` needs. |
| `wyze-scale-weight-tracker.json` | The n8n workflow, exported from the live instance with credentials and instance data stripped. Import it into n8n. |

---

## Requirements 📋

- **Python 3.12–3.14** with **`wyze-sdk` ≥ 2.3.8**, in a virtualenv on the server, and installed locally for the bootstrap script. Older `wyze-sdk` versions ignore the fetch window and download the scale's entire history on every run.
- **A self-hosted n8n** that can SSH into the server running the script. They can be the same machine.
- **A native Wyze account** with developer API keys.
- **A GitHub personal access token** with `repo` scope, plus a GitHub credential in n8n.

---

## First-Time Setup 🚀

The paths below (`/opt/tcs/scripts/…`, `/opt/tcs/n8n/.env`) are this setup's. If yours differ, change `ENV_FILE` at the top of both scripts, and the command in the workflow's **Execute Wyze Script** node.

### 1. Wyze prerequisites

- A **native** Wyze account. Google/Apple SSO accounts can't use the API.
- Developer API keys from [developer-api-console.wyze.com](https://developer-api-console.wyze.com)
- The Wyze Scale must be owned by this API account, or shared with it and accepted.
- Step on the scale at least once after linking, so there's a measurement record.

### 2. Put the script on the server

On the server, create the folder and give the script its own virtualenv (one time):

```bash
sudo mkdir -p /opt/tcs/scripts && sudo chown "$USER" /opt/tcs/scripts
python3 -m venv /opt/tcs/scripts/wyze-env
/opt/tcs/scripts/wyze-env/bin/pip install "wyze-sdk>=2.3.8"
```

Then, from your machine:

```bash
scp get_wyze_data.py you@your-server:/opt/tcs/scripts/
```

### 3. Set up the server `.env`

`get_wyze_data.py` reads, and rewrites, `/opt/tcs/n8n/.env` (its `ENV_FILE`). Add your Wyze developer keys:

```
WYZE_KEY_ID=<your key ID>
WYZE_API_KEY=<your API key>
```

The bootstrap in step 5 adds `WYZE_ACCESS_TOKEN` and `WYZE_REFRESH_TOKEN`.

**The GitHub token goes to n8n, not the script.** The **Get File** node reads `GITHUB_PAT` from n8n's own environment, so the n8n container needs:
- `GITHUB_PAT=<token with repo scope>`
- `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, so nodes can read `$env`

In this setup, the same `.env` is the n8n compose env file, and compose passes `GITHUB_PAT` into the container.

> The workflow's sticky note also lists `WYZE_PHONE_ID`. Nothing reads it any more, so you can skip it.

### 4. Local credentials file

```bash
cp credentials.example.py credentials.py
```

Edit `credentials.py` with your real values:

```python
EMAIL    = 'your-wyze-email@example.com'
PASSWORD = 'your-wyze-password'
KEY_ID   = 'your-key-id-from-wyze-developer-portal'
API_KEY  = 'your-api-key-from-wyze-developer-portal'
SERVER   = 'you@your-server.example.com'
```

`SERVER` must be a login that can write the server's `.env`.

> `credentials.py` is listed in `.gitignore` and will never be committed.

### 5. Bootstrap the tokens

Install `wyze-sdk` locally (one time), then run the bootstrap from your local machine. The login needs a residential IP.

```bash
pip install "wyze-sdk>=2.3.8"
python bootstrap-wyze-tokens.py
```

This logs in and prints two commands that set the token lines, replacing them if they exist and adding them if not. Run them in a terminal on the server. Or re-run with `--push` to have the script SSH in and apply them itself; that needs key-based SSH, since it runs non-interactively.

```bash
python bootstrap-wyze-tokens.py --push
```

### 6. Test it on the server

**Heads-up:** this is a real run. It refreshes the tokens and rewrites `.env`, exactly like the scheduled run does. That's safe; it's how the chain works.

```bash
/opt/tcs/scripts/wyze-env/bin/python /opt/tcs/scripts/get_wyze_data.py
```

Expected output:

```json
{"data": {"weight": 211.6, "body_fat": 27.2, "bmi": 27.7, "bmr": 1879.0, "muscle": 65.5, ...}}
```

### 7. Import the workflow

Import `wyze-scale-weight-tracker.json` into n8n. Then:

1. **Execute Wyze Script:** attach an SSH credential (private key) for your server, and check the command's paths.
2. **Get File:** change the URL to your own tracking file, e.g. `https://api.github.com/repos/YOUR_USER/YOUR_REPO/contents/health/weight-tracking.md`
3. **Edit a file:** attach your GitHub credential, and set the owner, repository and file path to the same file.
4. Activate the workflow.

**The tracking file must end with a markdown table.** The workflow appends each row to the end of the file, and the table needs these 11 columns:

```
| Date | Weight | BMI | Body Fat | Muscle | Body Water | Bone Mineral | Protein | Metabolic Age | BMR | Notes |
|------|--------|-----|----------|--------|------------|--------------|---------|---------------|-----|-------|
```

---

## Normal Operation ✅

Once set up, nothing needs to be touched. Every run refreshes the tokens and writes them back to `.env`, so the chain stays live as long as the workflow runs at least once every 28 days. It runs daily.

**What gets logged:**
- **Each run logs at most one reading:** the newest one, and only if it's from today and hasn't been logged yet.
- **Readings after 10:00:** by the next morning they're "yesterday" and get skipped, so run the workflow manually from the n8n editor to log them.
- **Days with no weigh-in:** nothing is logged, and the run still succeeds.

**When a run fails**, it fails in **Parse Python Output**. The SSH node passes the script's output along even when the script exits with an error, and the error message says why:
- *"No records returned"*: no reading in the last 30 days, or the scale isn't shared with the API account.
- *"get_records failed"*: a Wyze API error.
- A failed token refresh is only a warning. The script carries on with the existing token.

### Rotating the Wyze developer key

1. Generate a new Key ID and API Key at [developer-api-console.wyze.com](https://developer-api-console.wyze.com).
2. Update `WYZE_KEY_ID` and `WYZE_API_KEY` in the server `.env`.

No restart is needed, because the script reads `.env` on every run.

---

## 🚨 Break-Glass: Token Chain Broken

If the workflow hasn't run for more than 28 days, the refresh token will have expired. Fix it by running the bootstrap again from your local machine:

```bash
python bootstrap-wyze-tokens.py --push
```

This does a fresh login from your residential IP, bypassing the datacenter block, and pushes new tokens to the server. No n8n restart is needed.

---

## What Gets Logged 📊

Each row is built by **Append New Info**:

| Column | Value |
|--------|-------|
| Date | Date and time of the reading in `America/Toronto`, e.g. `2026-09-29 07:42:10` |
| Weight | lbs, rounded to a whole number |
| BMI | one decimal |
| Body Fat | % |
| Muscle | % |
| Body Water | % |
| Bone Mineral | as reported by the scale |
| Protein | % |
| Metabolic Age | years |
| BMR | kcal, rounded |
| Notes | left empty, for your own notes |

Body-composition values need the scale's impedance measurement. When the scale only captures weight (stepped off too fast, wet feet), the values come back empty or `0`, and the cells are left blank. The `%` columns show a bare `%`.

### Everything the script returns

`get_wyze_data.py` prints every field on the newest `wyze-sdk` `ScaleRecord`. The workflow uses the ones above, plus `measure_ts` for the date check and the dedup check.

| Field | Notes |
|-------|-------|
| `weight` | Already in **lbs**; `wyze-sdk` converts it |
| `body_fat`, `muscle`, `body_water`, `protein` | Percentages |
| `bmi`, `bmr`, `bone_mineral`, `metabolic_age` | As logged above |
| `body_vfr` | Visceral fat rating (not logged) |
| `measure_ts` | Unix timestamp, milliseconds |
| `timezone` | e.g. `America/Toronto` |
| `age`, `height`, `gender`, `body_type`, `occupation` | Profile fields; some are often empty |
| `impedance`, `measure_type`, `attributes` | Raw measurement details |
| `id`, `device_id`, `mac`, `user_id`, `family_member_id` | Record, device and account identifiers |

---

## Architecture Notes 🏗️

- **Why SSH instead of a direct HTTP call?** Wyze's internal scale API requires HMAC-MD5 signed requests with a hardcoded signing secret. The `wyze-sdk` Python library is far simpler and more maintainable than reimplementing that signing in JavaScript inside an n8n Code node.
- **Why not store the tokens in n8n credentials?** The tokens rotate on every run, and n8n's credential store can't be updated from inside a workflow. Instead, the script writes the new tokens back to the same `.env` file it reads them from.
- **Weight is already in lbs.** `wyze-sdk` converts from the scale's raw kg value. Don't multiply by 2.20462 anywhere; **Parse Python Output** relies on this.
- **Two timezones to change if you're elsewhere:**
  - the schedule, which uses n8n's `GENERIC_TIMEZONE`
  - the "is this reading from today?" check, which is hard-coded to `America/Toronto` in **Parse Python Output**
- **Why a 30-day fetch window?** Only the newest record is used, and the workflow checks its date itself. The window just has to be wide enough to always contain *a* reading. A narrow one would turn "didn't weigh in for a couple of days" into an error.

---

## 🧡 Support

This tool is free and always will be. If it saves you from another manual weight log, consider supporting on Patreon — it's how the tools and automation experiments keep coming.

[![Support on Patreon](https://raw.githubusercontent.com/Godimas101/personal-projects/main/patreon/images/buttons/patreon-medium.png)](https://patreon.com/Godimas101)

---

*"Automate the boring stuff — especially the part where you write down your weight."*
