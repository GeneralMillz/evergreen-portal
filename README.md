# evergreen-portal

**Private repository — managed by Evergreen OS running on a Raspberry Pi 5.**

This repo is the delivery mechanism for AI-generated website portal previews built by the Evergreen OS lead generation pipeline. Each subdirectory is one business's portal: a complete, self-contained website preview generated from their Google Business Profile data, SEO analysis, and competitor intelligence.

No manual commits happen here. Everything is written, committed, and pushed automatically by `portal_publisher.py` during Swarm Phase 3.

---

## What this repo is for

When Evergreen OS identifies a qualified local business lead, it automatically:

1. Builds a full website preview for the business using **SiteKit** (the AI site builder).
2. Generates a **portal** — a landing page that the business owner can view and see what their new site would look like.
3. Publishes the portal output here so it can be served via GitHub Pages or a static host and linked in the outreach email sent to the business.

The outreach email says something like:
> "I built you a free preview of what your site could look like. Take a look: https://generalmillz.github.io/evergreen-portal/great_cuts/portal/"

---

## Repository structure

```
evergreen-portal/
├── README.md
├── .gitignore
│
├── <business_slug>/           ← one directory per business lead
│   ├── index.html             ← main website preview
│   ├── portal/
│   │   └── index.html         ← portal landing page (linked in outreach email)
│   ├── assets/
│   │   ├── style.css
│   │   └── ...
│   ├── images/
│   │   └── ...
│   └── scripts/
│       └── ...
│
├── great_cuts/
│   ├── index.html
│   ├── portal/index.html
│   └── assets/...
│
├── b_pitters_painting_inc/
│   └── ...
│
└── ...
```

### Slug format

Business names are normalized to a URL-safe slug:
- Lowercase
- Non-alphanumeric characters stripped (except hyphens)
- Spaces and separators → underscores
- Max 40 characters

Examples:
| Business Name | Slug |
|---|---|
| Great Cuts | `great_cuts` |
| B. Pitter's Painting Inc. | `b_pitters_painting_inc` |
| Gold-N Bakery | `gold-n_bakery` |

---

## How Evergreen OS uses this repo

### Pipeline overview

```
Lead Scanner (Node.js)
    ↓  data/leads.json
Lead Refresh Engine (Python)
    ↓  competitor + review intel enrichment
Swarm (Python, multi-agent)
    Phase 1  → CEO initialization
    Phase 2  → Lead scoring + top-150 selection
    Phase 2.5 → Qualification filter (composite_score ≥ 40)
    Phase 3  → Per-lead pipeline:
        1. Email enrichment (verify email exists)
        2. SiteKit build (import_lead + portal)        ← writes to sitekit/projects/<slug>/
        3. Portal URL assignment                        ← http://localhost:8100/portals/<slug>/portal/
        4. portal_publisher.publish_to_portal_repo()   ← commits + pushes here
        5. Outreach draft generation
    Phase 4  → Quality analysis
    Phase 5  → CEO review + approval
```

### What portal_publisher.py does

Located at: `swarm/tools/portal_publisher.py`

1. Clones this repo to `/tmp/evergreen-portal-staging` on first run; pulls `--ff-only` on subsequent runs.
2. For each project slug, copies only these items from the SiteKit output directory into `<clone>/<slug>/`:
   - `index.html`
   - `assets/`
   - `images/`
   - `scripts/`
   - `portal/`
3. Runs `git add <slug>`, `git commit -m "Publish portal for <slug>"`, `git push origin main`.
4. If nothing changed (already up to date), skips the commit gracefully.
5. Never touches any file outside the project's own subdirectory.
6. Returns `True` on success, `False` on failure (logs at ERROR level — never raises).

### Source and destination paths

| Item | Path |
|---|---|
| SiteKit output (source) | `/mnt/storage/pi-assistant/skills/evergreen/sitekit/projects/<slug>/` |
| Publisher staging clone | `/tmp/evergreen-portal-staging/` |
| GitHub destination | `git@github.com:GeneralMillz/evergreen-portal.git` → `main` branch |

---

## How to manually inspect portal output

### Check what's been published

```bash
# List all published projects
ls /tmp/evergreen-portal-staging/

# Or clone fresh
git clone git@github.com:GeneralMillz/evergreen-portal.git /tmp/portal-check
ls /tmp/portal-check/
```

### View a portal locally

```bash
# Open a specific portal in browser (on Pi)
python3 -m http.server 9000 --directory /mnt/storage/pi-assistant/skills/evergreen/sitekit/projects
# Then visit: http://localhost:9000/<slug>/portal/
```

### Check the SiteKit source files exist

```bash
# Verify a project was built
ls /mnt/storage/pi-assistant/skills/evergreen/sitekit/projects/<slug>/

# Expected output:
# index.html  portal/  assets/  images/  site_settings.json  build_quality.json
```

---

## How to verify a successful publish

```bash
# 1. Check the staging clone for the project
ls /tmp/evergreen-portal-staging/<slug>/portal/

# 2. Check git log for the commit
git -C /tmp/evergreen-portal-staging log --oneline -5

# 3. Check GitHub directly
gh repo view GeneralMillz/evergreen-portal --web
# Or via API:
gh api repos/GeneralMillz/evergreen-portal/git/refs/heads/main

# 4. Confirm the portal URL in the Swarm draft
cat /mnt/storage/pi-assistant/skills/evergreen/swarm/results/outreach_drafts.json \
  | python3 -c "import json,sys; [print(d['lead_name'], '→', d.get('portal_url','')) for d in json.load(sys.stdin)]"
```

---

## Verification plan — run this after the first campaign

```bash
# Step 1: Confirm SiteKit writes to the new unified directory
ls /mnt/storage/pi-assistant/skills/evergreen/sitekit/projects/

# Step 2: Test a single SiteKit build manually
echo '{"name":"Test Co","address":"1 Main St","phone":"555-0100"}' > /tmp/test_lead.json
cd /mnt/storage/pi-assistant/skills/evergreen/Evergreen-tools/sitekit
./run_sitekit.sh import_lead test_co --lead-file /tmp/test_lead.json
ls /mnt/storage/pi-assistant/skills/evergreen/sitekit/projects/test_co/

# Step 3: Generate the portal for that test project
./run_sitekit.sh portal test_co
ls /mnt/storage/pi-assistant/skills/evergreen/sitekit/projects/test_co/portal/

# Step 4: Run publisher manually on the test project
cd /mnt/storage/pi-assistant/skills/evergreen/swarm
source /mnt/storage/pi-assistant/venv/bin/activate
python3 -c "
from tools.portal_publisher import publish_to_portal_repo
result = publish_to_portal_repo('test_co')
print('Published:', result)
"

# Step 5: Verify it landed in GitHub
git -C /tmp/evergreen-portal-staging log --oneline -3

# Step 6: Run the full Swarm dry-run (no outreach generation, just scoring/analysis)
cd /mnt/storage/pi-assistant/skills/evergreen/swarm
python3 run.py --leads data/leads.json --dry-run

# Step 7: Run the full campaign
python3 run.py --leads data/leads.json
```

---

## Troubleshooting

### Publisher fails: "Project directory not found"

```
[publisher] Project directory not found: .../sitekit/projects/<slug> — skipping
```

**Cause:** SiteKit build failed or the slug computed by `sitekit_tools.py` doesn't match what `portal_publisher.py` expects.

**Fix:**
```bash
# Verify the slug that was computed
python3 -c "import re; n='Business Name'; s=re.sub(r'[^\w\s-]','',n.lower()); s=re.sub(r'[\s_-]+','_',s).strip('_'); print(s[:40])"

# Check if the directory exists under that slug
ls /mnt/storage/pi-assistant/skills/evergreen/sitekit/projects/
```

### Publisher fails: "Clone failed"

```
[publisher] Clone failed: ...
```

**Cause:** SSH key not loaded, or repo URL wrong.

**Fix:**
```bash
# Test SSH auth to GitHub
ssh -T git@github.com

# Verify remote URL
git -C /tmp/evergreen-portal-staging remote -v
# Should show: git@github.com:GeneralMillz/evergreen-portal.git
```

### Publisher fails: "git push failed"

```
[publisher] git push failed: ...
```

**Cause:** Usually a diverged history (someone pushed manually) or SSH issue.

**Fix:**
```bash
# Nuke the staging clone and let it re-clone fresh
rm -rf /tmp/evergreen-portal-staging
# Then re-run the publisher — it will re-clone automatically
```

### SiteKit still writes to old path

**Symptom:** New projects appear in `/mnt/storage/projects/websitework/` instead of the new path.

**Fix:** Verify `run_sitekit.sh`:
```bash
cat /mnt/storage/pi-assistant/skills/evergreen/Evergreen-tools/sitekit/run_sitekit.sh
# Must show: cd /mnt/storage/pi-assistant/skills/evergreen/sitekit/projects
```

### No portal files to publish (nothing stageable)

```
[publisher] Nothing stageable found for '<slug>' — skipping push
```

**Cause:** SiteKit build ran but the portal step failed. The publisher only copies `index.html`, `assets/`, `images/`, `scripts/`, `portal/` — if none of those exist, nothing is staged.

**Fix:**
```bash
# Run the portal step manually
cd /mnt/storage/pi-assistant/skills/evergreen/Evergreen-tools/sitekit
./run_sitekit.sh portal <slug>
ls /mnt/storage/pi-assistant/skills/evergreen/sitekit/projects/<slug>/portal/
```

---

## Operational Readiness Report

| Component | Status | Path |
|---|---|---|
| SiteKit launcher (`run_sitekit.sh`) | ✅ Writes to unified path | `sitekit/projects/<slug>/` |
| `swarm_bridge.py` `WEBSITEWORK` | ✅ Updated | `sitekit/projects` |
| `swarm_outreach_runner.py` `WEBSITEWORK` | ✅ Updated | `sitekit/projects` |
| `email_enricher.py` `WEBSITEWORK` | ✅ Updated | `sitekit/projects` |
| `portal_publisher.py` `WEBSITEWORK_DIR` | ✅ Updated | `sitekit/projects` |
| Legacy path in `.py`/`.sh` files | ✅ None remaining | — |
| Legacy path in `.json`/`.md` (data) | ⚠️ 2 files, historical only | Non-functional |
| Publisher staging dir | ✅ `/tmp/evergreen-portal-staging` | Auto-created on first run |
| GitHub remote | ✅ `git@github.com:GeneralMillz/evergreen-portal.git` | `main` branch |
| Phase 2.5 qualification | ✅ 150 leads flow to Phase 3 | score ≥ 40, require_email=false |
| Swarm Phase 3 portal block | ✅ After email enrichment | Only email-verified leads get SiteKit builds |
| Outreach drafts output | ✅ `swarm/results/outreach_drafts.json` | — |

**Remaining risks:**
- The `/tmp/evergreen-portal-staging` clone persists across runs but is in `/tmp` — a system reboot clears it. The publisher re-clones automatically; no data loss, just a ~10s delay on the first run after reboot.
- Existing projects in `/mnt/storage/projects/websitework/` (29 projects built before the path fix) will not be re-published unless manually re-run. They are not deleted — they simply won't appear in new campaigns.
- `portal_tools.py` returns `http://localhost:8100/portals/<slug>/portal/` — this URL is included in outreach drafts as the local preview URL. For the publicly shareable URL (GitHub Pages), configure GitHub Pages on the `evergreen-portal` repo and update `PORTAL_BASE_URL` in `portal_tools.py`.

---

*This repository is managed by Evergreen OS — an autonomous lead generation pipeline running on a Raspberry Pi 5. Do not commit to this repo manually unless you know what you are doing. Manual commits will not break anything but may cause a pull conflict on the next automated push.*
