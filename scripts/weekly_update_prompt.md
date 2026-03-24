# Weekly Dashboard Update — Automated Agent Prompt

Run every Tuesday at 7am AEDT (Monday 8pm UTC). This agent fetches the latest weekly data from Notion and updates the dashboard.

---

## Step 1: Fetch the current week's Notion data

Use `mcp__e6d98c4c-afc0-4630-a7bb-2ebf642373b6__notion-fetch` to fetch the most recent entry in the **True Attribution Analysis** Notion database.

The most recent week entry URL follows this pattern: fetch the page and read these **direct numeric** properties:
- `Week Name` — e.g. "Mar 16-22, 2026"
- `Activation Rate` — decimal 0–1 scale (multiply × 100 for %)
- `Onboarding Completion Rate` — decimal 0–1 scale (multiply × 100 for %)
- `Weekly Acquistion link` — relation to the **Weekly Acquisition Snapshot** page (follow this link)
- `Previous Week` — relation to the prior week's True Attribution Analysis page (follow this link to compute WoW)

From the **Weekly Acquisition Snapshot** page, read:
- `Spend - Meta`, `Spend - Apple Ads`, `Spend - Google`
- `Installs - Meta`, `Installs - Apple Ads`, `Installs - Google`
- `Organic Installs`, `Organic trials`, `Organic new subs`
- `Trials - Meta`, `Trials - Apple Ads`, `Trials - Google`
- `Trial conversions - Meta`, `Trial conversions - Apple Ads`, `Trial conversions - Google`, `Trial conversions - Organic`
- `New subs - Meta`, `New subs - Apple Ads`, `New subs - Google`
- `Net Rev(b4spend) - Meta`, `Net Rev(b4spend) - Apple Ads`, `Net Rev(b4spend) - Google`
- `Net Rev - Organic`

Also fetch the **Previous Week** Acquisition Snapshot (via the Previous Week True Attribution Analysis page → its `Weekly Acquistion link`) to compute WoW % for all metrics.

---

## Step 2: Compute derived metrics

For each paid platform (meta, apple, google):
```
cpi    = spend / installs
cpt    = spend / trials
cac    = spend / new_subs
roas   = net_rev_b4spend / spend
```

Totals across all paid:
```
totalSpend   = meta.spend + apple.spend + google.spend
totalInstalls = meta.installs + apple.installs + google.installs
totalTrials  = meta.trials + apple.trials + google.trials
totalConversions = meta.trial_conversions + apple.trial_conversions + google.trial_conversions + organic.trial_conversions
totalNewSubs  = meta.new_subs + apple.new_subs + google.new_subs
totalNetRev  = meta.net_rev + apple.net_rev + google.net_rev

blendedNetRev  = totalNetRev + organic.net_rev
blendedNetROAS = (blendedNetRev + totalSpend) / totalSpend

paidInstalls   = totalInstalls
totalPaidInstalls = paidInstalls + organic.installs  (for display as "installs" on overview)
```

WoW % = `Math.round((current - prev) / prev * 100)` — set to `null` if no prior week.

For activation/onboarding WoW: compare current True Attribution Analysis `Activation Rate` / `Onboarding Completion Rate` to the Previous Week page's same fields.

---

## Step 3: Build the new DATA entry

Insert the new week's data as the **first key** in the `DATA` object in `index.html`. The key format is `"Mon DD-DD, YYYY"` matching the Notion `Week Name` field exactly.

Structure to insert (mirror existing weeks exactly):
```js
"[Week Name]": {
  meta: {
    spend: [Spend - Meta],
    installs: [Installs - Meta],
    trials: [Trials - Meta],
    trialConversions: [Trial conversions - Meta],
    newSubs: [New subs - Meta],
    netRev: [Net Rev(b4spend) - Meta],
    cpi: [meta.spend / meta.installs],
    cpt: [meta.spend / meta.trials],
    cac: [meta.spend / meta.newSubs],
    roas: [meta.netRev / meta.spend],
    cpiWoW: [WoW %],
    cptWoW: [WoW %],
    cacWoW: [WoW %],
    roasWoW: [WoW %]
  },
  apple: { /* same shape */ },
  google: { /* same shape */ },
  organic: {
    installs: [Organic Installs],
    trials: [Organic trials],
    newSubs: [Organic new subs],
    netRev: [Net Rev - Organic],
    trialConversions: [Trial conversions - Organic]
  },
  totalSpend: [sum],
  totalSpendWoW: [WoW %],
  blendedNetRev: [sum],
  blendedNetRevWoW: [WoW %],
  blendedNetROAS: [(blendedNetRev + totalSpend) / totalSpend],
  blendedNetROASWoW: [WoW %],
  spendWoW: [WoW %],
  revWoW: [WoW %],
  roasWoW: [WoW %],
  totalPaid: {
    installs: [totalPaidInstalls],
    installsWoW: [WoW %],
    trials: [totalTrials],
    trialsWoW: [WoW %],
    conversions: [totalConversions],
    conversionsWoW: [WoW %],
    subs: [totalNewSubs],
    subsWoW: [WoW %]
  },
  activationRate: [Activation Rate × 100],
  activationRateWoW: [WoW %],
  onboardingCompletionRate: [Onboarding Completion Rate × 100],
  onboardingCompletionRateWoW: [WoW %]
},
```

---

## Step 4: Update index.html

1. Open `/Users/lukelongworth/True_attribution_unscripted/index.html` (the main repo file, not the worktree).
2. Find the `const DATA = {` line.
3. Insert the new week's entry as the first key in the object.
4. The `selectedWeek` default value is the first key of `DATA` — no change needed if you insert at the top.

---

## Step 5: Commit, push, create PR, and merge

```bash
cd /Users/lukelongworth/True_attribution_unscripted
git checkout -b weekly/update-$(date +%Y-%m-%d)
git add index.html
git commit -m "Weekly data update: [Week Name]"
git push origin weekly/update-$(date +%Y-%m-%d)
gh pr create --title "Weekly data update: [Week Name]" --body "Automated weekly dashboard update. Data sourced from Notion True Attribution Analysis."
gh pr merge --squash --auto
```

The PR will auto-merge once created (no review required).
