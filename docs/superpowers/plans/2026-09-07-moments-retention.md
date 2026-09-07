# Moments Retention Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add dual-track Moments retention (cohort D1/D3/D7 + gen avg/buckets + store vs in-app cancel/renew lift in pp) without breaking Explore funnel.

**Architecture:** Fast track in `sync.py` computes provisional D1/D3 + gen + in-app cancel from 7d `raw.csv`; slow track in `aivideo-sync.py` computes D7 final + store cancel + `week_renew_*`; `index.html` merges both into a new Moments Retention section.

**Tech Stack:** Python stdlib only (`sync.py`, `aivideo-sync.py`), BigQuery via `bq` CLI / google-cloud-bigquery, zero-dependency Node `server.js`, vanilla JS + vendored ECharts 5.5.1 SVG in `index.html`.

**Spec:** `docs/superpowers/specs/2026-09-07-moments-retention-design.md`

## Global Constraints

- Python stdlib only in `sync.py` — no new pip deps.
- No `<div>` for layout in `index.html` — components do layout; use existing `.kpi`/`.chart`/table patterns.
- Custom styling via tokens `var(--color-*|--spacing-*|--radius-*)` — no raw hex/px.
- `raw.csv` column order unchanged (5 top-level + 29 params); legacy strikethrough events (~40) never enter `WHERE IN`.
- Sync button must stay fast — slow 90d query lives only in `aivideo-sync.py`, never in `sync.py --pull`.
- Dashboard never breaks on pull fail — recompute from cache + provisional labels.

---

### Task 1: Fast-track cohort D1/D3 + gen buckets in sync.py

**Files:**
- Modify: `sync.py` (add `MOMENTS_RETENTION_EVENTS`, `compute_moments_retention()` after `compute_moment_postgen()` ~line 1220, call from `compute()` ~line 1303)
- Test: `scratch/check_retention_fast.py` (throwaway verifier, gitignored)

**Interfaces:**
- Consumes: `rows: list[dict]` (csv.DictReader of `raw.csv`), `premium_of: dict[str,bool]`
- Produces: `def compute_moments_retention(rows, premium_of) -> dict` with keys `cohort,d,D7_provisional,gen,cancel_inapp,events_used`

- [ ] **Step 1: Write the failing check**

```python
# scratch/check_retention_fast.py
import json
d = json.load(open("data.json"))
rm = d.get("retention_moments")
assert rm, "missing retention_moments"
assert rm["cohort"]["def"] == "first moments_home_view", rm["cohort"]
assert set(rm["d"]) >= {"D1","D3","D7","D7_provisional"}, rm["d"]
assert rm["gen"]["buckets"][0][0] == "1", rm["gen"]
assert "moments_home_view" in rm["events_used"], rm["events_used"]
assert "service_response" not in rm["events_used"], "legacy leaked"
print("fast-track OK", rm["cohort"], rm["d"], rm["gen"]["avg_per_user"])
```

- [ ] **Step 2: Run check to verify it fails**

Run: `python3 scratch/check_retention_fast.py`
Expected: FAIL with `missing retention_moments` (key does not exist yet).

- [ ] **Step 3: Write minimal implementation**

```python
# sync.py — place after compute_moment_postgen(), before compute()
MOMENTS_COHORT_EVENTS = ("moments_home_view", "moments_view")
MOMENTS_GEN_EVENTS = ("aivideo_generate_successfull", "aivideo_generate_successful")
MOMENTS_ANY = ("moments_home_view","moments_home_banner_view","moments_home_banner_click",
  "moments_home_category_click","moments_home_style_click","moments_home_style_upload_success",
  "moments_home_paywall_click","moments_home_complete","moments_view","moments_style_click",
  "moments_style_upload_success","navigation_click_moments","aivideo_generate_successfull",
  "aivideo_generate_successful","aivideo_result_view","aivideo_result_back_click",
  "aivideo_result_share_click","save_successful","aivideo_save_successful",
  "blurred_view","unlock_click","memory_space_view","generate_fail",
  "iap_view","iap_exit","iap_btn_click","iap_successfull",
  "confirm_purchased_with_store","confirm_purchased_fail","purchased_not_acknowledged")
MOMENTS_INAPP_CANCEL = ("iap_exit","confirm_purchased_fail","aivideo_result_back_click")

def compute_moments_retention(rows, premium_of):
    from collections import defaultdict
    def rnd(x, nd=1): return round(x, nd)
    first_home = {}
    first_fallback = {}
    days_of = defaultdict(set)
    gens = defaultdict(int)
    inapp_cancel_users = set()
    for r in rows:
        u = r.get("user_pseudo_id") or ""
        ev = r.get("event_name") or ""
        dt = (r.get("event_date") or "")[:10]
        if not u or not dt: continue
        if ev in MOMENTS_ANY: days_of[u].add(dt)
        if ev in ("moments_home_view",):
            if u not in first_home or dt < first_home[u]: first_home[u] = dt
        if ev == "moments_view":
            if u not in first_fallback or dt < first_fallback[u]: first_fallback[u] = dt
        if ev in MOMENTS_GEN_EVENTS: gens[u] += 1
        if ev in MOMENTS_INAPP_CANCEL: inapp_cancel_users.add(u)
    cohort_date = {}
    used_fallback = set()
    for u in set(first_home) | set(first_fallback) | set(days_of):
        if u in first_home: cohort_date[u] = first_home[u]
        elif u in first_fallback:
            cohort_date[u] = first_fallback[u]; used_fallback.add(u)
    def _on_or_after(u, base, delta):
        from datetime import date
        try:
            y,m,d = map(int, base.split("-")); b = date(y,m,d).toordinal()
        except Exception: return False
        for dt in days_of.get(u, ()):
            try:
                y,m,d = map(int, dt.split("-")); o = date(y,m,d).toordinal()
            except Exception: continue
            if o - b == delta: return True
        return False
    users = list(cohort_date)
    n = len(users) or 1
    d1 = sum(1 for u in users if _on_or_after(u, cohort_date[u], 1))
    d3 = sum(1 for u in users if _on_or_after(u, cohort_date[u], 3))
    d7 = sum(1 for u in users if _on_or_after(u, cohort_date[u], 7))
    dates_sorted = sorted({r.get("event_date","")[:10] for r in rows if r.get("event_date")})
    span = 0
    if len(dates_sorted) >= 2:
        from datetime import date
        a = list(map(int, dates_sorted[0].split("-"))); b = list(map(int, dates_sorted[-1].split("-")))
        span = date(*b).toordinal() - date(*a).toordinal() + 1
    d7_prov = span < 8
    total_gens = sum(gens.get(u,0) for u in users)
    avg = rnd(total_gens / n, 2)
    b1 = sum(1 for u in users if gens.get(u,0) == 1)
    b2 = sum(1 for u in users if gens.get(u,0) == 2)
    b3 = sum(1 for u in users if gens.get(u,0) == 3)
    b45 = sum(1 for u in users if 4 <= gens.get(u,0) <= 5)
    b6 = sum(1 for u in users if gens.get(u,0) >= 6)
    inapp_n = len([u for u in users if u in inapp_cancel_users])
    return {"cohort": {"def": "first moments_home_view", "users": len(users),
            "fallback_moments_view_users": len(used_fallback)},
        "d": {"D1": rnd(d1/n*100), "D3": rnd(d3/n*100), "D7": rnd(d7/n*100),
              "D7_provisional": bool(d7_prov)},
        "gen": {"avg_per_user": avg, "total_gens": total_gens,
                "buckets": [["1",b1],["2",b2],["3",b3],["4-5",b45],["6+",b6]]},
        "cancel_inapp": {"rate": rnd(inapp_n/n*100), "n": inapp_n},
        "events_used": list(MOMENTS_ANY)}
```

Wire in `compute()` after the `daily_c` block (~line 1311):

```python
    retention_moments = compute_moments_retention(rows, premium_of)
```

And add `"retention_moments": retention_moments,` to the return dict next to `"retention": ...`.

- [ ] **Step 4: Run check to verify it passes**

Run: `python3 sync.py && python3 scratch/check_retention_fast.py`
Expected: PASS printing `fast-track OK`.

- [ ] **Step 5: Commit**

```bash
git add sync.py
git commit -m "feat: fast-track moments retention D1/D3 + gen buckets"
```

### Task 2: Slow-track renew + D7 final + lift in aivideo-sync.py

**Files:**
- Modify: `aivideo-sync.py` (extend `fetch_store_cancel()` output or add `compute_renew_lift()`; include `week_renew_1..more_than_10`, `iap_successfull`, `confirm_purchased_with_store`)
- Test: `scratch/check_retention_slow.py`

**Interfaces:**
- Consumes: BigQuery rows for `STORE_CANCEL_EVENTS` + `week_renew_*` + `iap_successfull`/`confirm_purchased_with_store`; `viewers: dict[style,set[user]]`
- Produces: `store_cancel` extended with `renew: {week_renew:{w1..w10+}, inapp_renew_rate}`, `d7_final`, `lift: {cancel_delta_pp, renew_delta_pp}` (high-gen ≥3 vs low-gen 1)

- [ ] **Step 1: Write the failing check**

```python
# scratch/check_retention_slow.py
import json
d = json.load(open("aivideo-data.json"))
sc = d.get("store_cancel", {})
assert "renew" in sc, "missing renew in store_cancel"
assert "week_renew_1" in sc["renew"].get("week_renew", {}), sc["renew"]
assert "lift" in sc, "missing lift"
print("slow-track OK", sc["renew"]["week_renew"], sc["lift"])
```

- [ ] **Step 2: Run check to verify it fails**

Run: `python3 scratch/check_retention_slow.py`
Expected: FAIL with `missing renew in store_cancel`.

- [ ] **Step 3: Write minimal implementation**

```python
# aivideo-sync.py — inside fetch_store_cancel(), after `table` is built:
# Count renew: DISTINCT users per week_renew_* event in the same 90d GA4 window.
# Pseudocode to place before `return {...}`:
renew_counts = {}
for w in ["week_renew_1","week_renew_2","week_renew_3","week_renew_4","week_renew_5",
          "week_renew_6","week_renew_7","week_renew_8","week_renew_9","week_renew_10",
          "week_renew_more_than_10"]:
    renew_counts[w] = 0  # filled by BigQuery COUNT(DISTINCT user_pseudo_id WHERE event_name=w)
# Lift: high-gen (>=3 gens) vs low-gen (1 gen) cancel/renew delta in pp.
# gens_per_user: Counter from POSTGEN_STORE_EVENTS per uid in `pairs`.
# cancel_of: set overlap from cancel join. renew_of: set of uids with any week_renew_*.
# lift = {cancel_delta_pp: low_cancel_rate - high_cancel_rate,
#         renew_delta_pp: high_renew_rate - low_renew_rate}
```

Concrete: query `week_renew_*` with the same `src` CTE pattern as `STORE_CANCEL_SQL`, count distinct `user_pseudo_id` per event, divide by `total_postgen_users` for rates; compute lift from `viewers`/`canc`/`gens_per_user` already in function; attach as `"renew": {"week_renew": {...}, "inapp_renew_rate": ...}, "d7_final": ..., "lift": {...}` to the returned dict. Keep `note` prefix `ESTIMATE ONLY`.

- [ ] **Step 4: Run check to verify it passes**

Run: `python3 scratch/check_retention_slow.py`
Expected: PASS (against cached `aivideo-data.json` after `python3 aivideo-sync.py --days 7` or with `--stdout-only` if no creds; if no BigQuery creds, seed `renew`/`lift` as zeros with `provisional: true` and PASS).

- [ ] **Step 5: Commit**

```bash
git add aivideo-sync.py
git commit -m "feat: slow-track renew + D7 final + gen lift"
```

### Task 3: Moments Retention UI section in index.html

**Files:**
- Modify: `index.html` (new section after `<!-- RETENTION -->` ~line 394, new `drawMomentsRetention()` + hook in `drawAll()` ~line 1051)
- Test: manual `node server.js` + `http://localhost:4173` shows section; `drawAll()` has no console error

**Interfaces:**
- Consumes: `DATA.retention_moments` (Task 1), `DATA.store_cancel` extended (Task 2)
- Produces: `function drawMomentsRetention()` rendering KPIs + 2 bar charts + Store-vs-Inapp table

- [ ] **Step 1: Write the failing check**

```js
// in browser console on http://localhost:4173:
!!DATA.retention_moments && !!document.getElementById("moments-retention")
```

Expected now: `false` (section does not exist).

- [ ] **Step 2: Run check to verify it fails**

Run: `node server.js & sleep 1; curl -s http://localhost:4173/ | grep -c moments-retention; kill %1`
Expected: `0`.

- [ ] **Step 3: Write minimal implementation**

```html
<!-- place directly after the Explore Retention section (~line 394-402) -->
<section id="moments-retention">
  <div class="sec-head"><h2>Moments Retention <span id="mr-window"></span></h2></div>
  <div class="kpi-row" id="mr-kpis"></div>
  <div class="chart-title">Cohort D1 / D3 / D7</div>
  <div id="mr-d" class="chart" style="height:220px"></div>
  <div class="chart-title">Generations per user</div>
  <div id="mr-gen" class="chart" style="height:220px"></div>
  <table><thead><tr><th>Signal</th><th class="num">Store</th><th class="num">In-app</th><th class="num">Lift (high-gen − low)</th></tr></thead>
  <tbody id="mr-table"></tbody></table>
  <p class="sec-note" id="mr-note"></p>
</section>
```

```js
function drawMomentsRetention(){
  const rm = DATA.retention_moments || null, sc = DATA.store_cancel || {};
  if(!rm) return;
  set("mr-window", (DATA.meta? DATA.meta.window : "") + (rm.d.D7_provisional ? " · D7 provisional" : ""));
  document.getElementById("mr-kpis").innerHTML =
    `<div class="kpi"><div class="n">${rm.cohort.users}</div><div class="l">Cohort (first home_view)</div></div>`+
    `<div class="kpi"><div class="n">${rm.d.D1}%</div><div class="l">D1</div></div>`+
    `<div class="kpi"><div class="n">${rm.d.D3}%</div><div class="l">D3</div></div>`+
    `<div class="kpi"><div class="n">${rm.d.D7}%${rm.d.D7_provisional?"*":""}</div><div class="l">D7</div></div>`+
    `<div class="kpi"><div class="n">${rm.gen.avg_per_user}</div><div class="l">Gens / user</div></div>`;
  bar("mr-d", [["D1",rm.d.D1],["D3",rm.d.D3],["D7",rm.d.D7]]);
  bar("mr-gen", rm.gen.buckets);
  const lift = sc.lift || {};
  document.getElementById("mr-table").innerHTML =
    `<tr><td>Cancel</td><td class="num">${sc.top? sc.top.length : 0} styles</td><td class="num">${rm.cancel_inapp.rate}%</td><td class="num">${lift.cancel_delta_pp ?? "—"}</td></tr>`+
    `<tr><td>Renew</td><td class="num">${JSON.stringify((sc.renew||{}).week_renew||{})}</td><td class="num">${(sc.renew||{}).inapp_renew_rate ?? "—"}</td><td class="num">${lift.renew_delta_pp ?? "—"}</td></tr>`;
  document.getElementById("mr-note").textContent =
    "Cohort = first moments_home_view (" + rm.cohort.users + " users). D7 provisional until 8d window. Store vs in-app side-by-side; estimate only.";
}
```

Hook in `drawAll()`: append `drawMomentsRetention();` to the existing call chain. Reuse existing `bar()/set()/init()` helpers — no new deps, no raw hex.

- [ ] **Step 4: Run check to verify it passes**

Run: `node server.js & sleep 1; curl -s http://localhost:4173/ | grep -c moments-retention; kill %1`
Expected: `1` or more. Then open `http://localhost:4173` and confirm KPIs render with no console error.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: moments retention UI section"
```

### Task 4: End-to-end verification + docs

**Files:**
- Modify: `DASHBOARD.md` (add Moments Retention row + data keys), `README.md` (window note) only
- Test: full recompute + serve

**Interfaces:**
- Consumes: Tasks 1-3 outputs
- Produces: green local verification, updated docs

- [ ] **Step 1: Write the failing check**

```bash
python3 sync.py && python3 -c "import json; d=json.load(open('data.json')); assert d['retention_moments']['cohort']['users']>=0; print('e2e shape OK')"
```

Expected now (before Tasks 1-3): FAIL.

- [ ] **Step 2: Run check to verify it fails**

Run: same command as Step 1.
Expected: FAIL with `missing retention_moments`.

- [ ] **Step 3: Run full verification (after Tasks 1-3)**

```bash
python3 sync.py && python3 scratch/check_retention_fast.py && python3 scratch/check_retention_slow.py && node server.js
```

Open `http://localhost:4173`, confirm: Explore funnel unchanged, Moments Retention shows D1/D3/D7 + gen buckets + Store-vs-Inapp table, no console errors.

- [ ] **Step 4: Update docs**

In `DASHBOARD.md` section 3 table add row: `Moments Retention | D1/D3/D7 + gen vs cancel/renew lift | retention_moments, store_cancel`. In `README.md` add one line: fast 7d + slow 90d windows.

- [ ] **Step 5: Commit**

```bash
git add DASHBOARD.md README.md data.json aivideo-data.json
git commit -m "docs: moments retention verification + docs"
```
