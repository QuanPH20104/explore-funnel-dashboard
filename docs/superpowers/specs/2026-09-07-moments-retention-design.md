# Moments Retention — Design (2026-09-07)

## 1. Goal
Update Moments retention to answer: how much does cancel rate drop (pp) and renew rate lift (pp),
linked via intermediate metrics (retention D1/D3/D7, gen counts) when direct cancel/renew attribution is weak.

## 2. Context
- Dashboard: Explore funnel + Moment aiVideoV520 post-gen cancel (`index.html`, `sync.py`, `aivideo-sync.py`).
- Current retention: `one_day vs two_plus` on 7d rolling, labeled directional only.
- Current cancel: estimate `post-gen users ∩ store cancels` from `IIP032_subcriptions_monitoring (cancel_at)`, 90d.
- No renew rate yet.
- Event source: `/Users/ducna/Downloads/Artimind IOS event tracking.xlsx` (34 sheets).
  Parsed to `scratch/moments_excel_*.json` by 5 parallel readers + `scratch/moments_retention_event_shortlist.json`.

## 3. Event shortlist (legacy excluded)
- Moments new UI (8, keep all): `moments_home_view`, `moments_home_banner_view/click`, `moments_home_category_click`, `moments_home_style_click`, `moments_home_style_upload_success`, `moments_home_paywall_click`, `moments_home_complete`.
- Renew (11): `week_renew_1..10`, `week_renew_more_than_10` (Did Renew + Adjust mapping, no cancel in sheet).
- IAP funnel (7): `iap_view`, `iap_exit`, `iap_btn_click`, `iap_successfull`, `confirm_purchased_with_store/fail`, `purchased_not_acknowledged`.
- Gen core (20): `navigation_click_moments`, `moments_view`, `moments_home_view`, `moments_home_style_click`, `moments_home_style_upload_success`, `moments_style_click`, `moments_style_upload_success`, `home_style_suggest_view/click`, `aivideo_generate_successful[l]`, `aivideo_result_view/back_click/share_click`, `save_successful[l]`, `blurred_view`, `unlock_click`, `memory_space_view`, `generate_fail`.
- Legacy excluded (~40, font.strike=True): `feature_tutorial_rate_*` (4), `service_quality.service_response`, `*_tpf_*`, `restore_*` old block, `regenerate_*` old, `fullscr_share/down_click`, `exit_popup_save_click`, `generate_image_size_status`, `result_loading_noti_click`, etc.

## 4. Decisions (approved)
- Cancel/renew: Option C — dual Store + In-app side-by-side.
  - Cancel store: `cancel_at ∩ post-gen users` per style. Cancel in-app proxy: `iap_exit` + `confirm_purchased_fail` + `aivideo_result_back_click`.
  - Renew store/adjust: `week_renew_*`. Renew in-app: `iap_successfull` + `confirm_purchased_with_store`.
- Cohort: Option A — first `moments_home_view` (fallback `moments_view` on old app, flagged).
- Window: Option C — dual-track. Fast 7d for funnel + provisional D/Gen, slow 28-90d for D7 final + cancel/renew.
- Gen counts: Option C — avg per user + buckets [1,2,3,4-5,6+].
- Architecture: Option 2 — dual-track (recommended, matches store_cancel pattern).

## 5. Architecture (dual-track)
- Fast track (`sync.py --pull`, 7d GA4 → `raw.csv` → `data.json`):
  cohort D1/D3 provisional + gen avg/buckets + in-app cancel proxy. D7 empty + `D7_provisional=true`.
- Slow track (`aivideo-sync.py`, 90d → `aivideo-data.json` + `store_cancel` passthrough):
  D7 final + store cancel + `week_renew_*` + in-app renew cross-check.
- UI merges both, labels each KPI window. Explore funnel untouched.

## 6. Data model (new `data.json.retention_moments`)
```
cohort: {def: first moments_home_view, window: 7d fast, users}
d: {D1, D3, D7, D7_provisional}
gen: {avg_per_user, total_gens, buckets: [[1,n],[2,n],[3,n],[4-5,n],[6+,n]]}
cancel: {store_rate, store_n, inapp_rate, inapp_n}
renew: {week_renew: {w1..w10+}, inapp_renew_rate}
lift: {high_gen(>=3)_vs_low_gen(1)_cancel_delta_pp, renew_delta_pp}
events_used: [...], meta: {fast_window, slow_window}
```
`lift` directly answers giam-huy / tang-gia-han in pp.

## 7. BigQuery flow
- Fast: one GA4 query 7d, WHERE event_name IN (shortlist, no legacy). Cohort = MIN(date of moments_home_view) per user. D1/D3 = EXISTS any Moments event on cohort_date+1/+3. Gen = COUNT(generate_success[l]) per user. In-app cancel = EXISTS iap_exit/back_click after last gen.
- Slow: separate 90d query (no raw.csv). Join post-gen users with cancel_at + week_renew rates by original_transaction_id + iap_successfull/confirm_with_store.
- Guards: BQ_MAX_BYTES cap, pull fail → recompute cache + provisional label, never break dashboard.

## 8. UI (`index.html`)
- New `Moments Retention` section after old Retention (old kept, relabeled Explore).
- KPI row: cohort users, D1/D3/D7 (provisional badge), gen/user, -Xpp cancel, +Ypp renew.
- Charts: D-cohort bars, gen bucket bars (highlight high-gen).
- Table: Store vs In-app cancel/renew side-by-side + windows + events_used + estimate note.
- `drawAll()` += `drawMomentsRetention()`.

## 9. Errors / Testing / Rollout
- Errors: no moments_home_view → fallback moments_view + flag; no week_renew in GA4 → in-app only + `no store signal`; bytes over cap → abort slow, keep fast.
- Tests: recompute from cached raw.csv yields new shape; legacy absent from events_used; D7 provisional when window<8d; drawAll merge safe.
- Rollout: slow track first (no UI risk), then fast + UI. Success = readable -Xpp/+Ypp by gen.

## 10. Files
- Reads: `Artimind IOS event tracking.xlsx`, `sync.py`, `aivideo-sync.py`, `index.html`, `data.json`.
- Writes (impl later, not this spec): `sync.py` (retention_moments provisional), `aivideo-sync.py` (renew+D7), `index.html` (section), `data.json`/`aivideo-data.json` shapes.
- Artifacts now: `scratch/moments_excel_*.json`, `scratch/moments_retention_event_shortlist.json`.
