# Shared Pre-Flight Setup

**Read this BEFORE running any process test document.**
Time required: ~10 minutes the first time, ~2 minutes after that.

---

## Step 1 — Open all 4 tabs

Open these 4 URLs in **separate browser tabs** (not windows). All testing
documents assume you have all 4 open and logged in.

| # | Tab name | URL | What you should see |
|---|---|---|---|
| 1 | **Console Dash** | https://console.m.frappe.cloud/dash | The shop-floor entry app — list of process tiles (Master Batch Mixing, Compound Inspection, etc.) |
| 2 | **Console Desk** | https://console.m.frappe.cloud/app | The Frappe admin desk — sidebar with "Console Inspection", "Console Transaction Log" etc. |
| 3 | **Ops** | https://newspp.m.frappe.cloud/app | The Ops ERPNext desk — sidebar with "Stock Entry", "Quality Inspection", "MES Process Log" |
| 4 | **Legacy** | https://sppmaster.frappe.cloud/app | The Legacy ERPNext desk — sidebar with "Compound Inspection", "Delivery Challan Receipt", "Inspection Entry" |

✅ **Pass:** All 4 tabs load and show their respective home screens.

❌ **Fail action:** If any tab shows a login screen, log in with your usual
credentials (work email + password). If a tab shows an error or "Page Not
Found", file a 🔴 bug — staging may be down.

---

## Step 2 — Verify the deployed version is current

This is the most important pre-flight check. The recent deployment changed
the Console Inspection architecture; if your browser cache is stale, you'll
test the OLD UI by mistake.

### On the **Console Dash** tab:
1. Hard-refresh the page: **Cmd+Shift+R** (Mac) / **Ctrl+Shift+R** (Windows)
2. Click the **Compound Deviation** tile
3. The page should load without a JavaScript console error
   (To check: right-click → Inspect → Console tab. Should show no red errors.)

✅ **Pass:** Compound Deviation page loads cleanly.

❌ **Fail action:** Hard-refresh again. If the error persists, file a 🔴 bug
with a screenshot of the browser console (the red errors).

### On the **Console Desk** tab:
1. In the search bar (top), type: `Console Inspection`
2. You should see **"Console Inspection"** in the dropdown (this is the NEW
   unified doctype that just got deployed)
3. Click it → the listview should open

✅ **Pass:** "Console Inspection" appears in search and opens an empty (or
near-empty) listview.

❌ **Fail action:** If you only see the OLD "Console Compound Inspection" /
"Console Compound Deviation" but not "Console Inspection", the deployment
hasn't reached this site yet. **Wait 5-10 more minutes** for Frappe Cloud
to finish the build, then retry. If it still doesn't appear after 30 minutes,
file a 🔴 bug.

### On the **Legacy** tab:
1. In the search bar, type: `Legacy Bridge Log`
2. Open the listview
3. Find the most recent entry (top of the list)
4. Open it
5. Scroll to the bottom — you should see a field called **"Transformed Data"**
   showing JSON output

✅ **Pass:** Recent Legacy Bridge Log exists and shows transformed data.

⚠️ **Note:** The Bridge fix from today eliminates the "Stock Entry reference
not found" error. If you see recent Bridge Logs with that error from BEFORE
today's deployment, those are old failures — not a problem.

---

## Step 3 — Confirm your test identity

Each test scenario needs a real employee scan. You'll use these IDs across
all process tests:

| Role | Employee ID format | Example | Where to find |
|---|---|---|---|
| **Mixing Operator** | `HR-EMP-XXXXX` | (your usual operator badge) | Your physical employee card |
| **Compound Inspector** | `HR-EMP-XXXXX` | (Lab Assistant / R&D Assistant designation) | Your physical employee card |
| **Quality Manager** | `HR-EMP-XXXXX` | (Quality Manager designation) | Your QM's physical card |

### Verify the IDs work:

1. On **Console Dash** → click any process tile that asks for an employee scan
2. Type your Mixing Operator ID into the scan field → press Enter
3. Should show ✅ green checkmark with your name

❌ **Fail action:** If your ID shows "not authorized" or "designation not
mapped", check **Console Desk → Console Employee** to confirm the record
exists and has the right designation. If missing, ask your supervisor to
sync employee data — don't proceed.

---

## Step 4 — Identify a real test batch (per process)

Since staging is a production copy, you'll work with **real items from the
factory's actual catalog**. Pick a batch that's safe to test with:

### For Compound Inspection / Compound Deviation testing:
- Use a **real compound batch currently in the "Compound Inspection - SPP"
  warehouse** on staging
- Find one via Console Desk → Console Stock Cache → filter
  `warehouse = "Compound Inspection - SPP"` AND `actual_qty > 0`
- 📝 Write down the **batch_no** and **mix_barcode** — you'll use them
  throughout the test

### For Master Batch / Final Batch Mixing testing:
- Pick a **real Master Batch item code** the factory mixes regularly
  (e.g., MB_6122, MB_EH0039 — these are real production items)
- Use a small produced quantity (e.g., 0.05 - 0.1 kg) so you don't tie up
  significant raw material stock on staging

### For Sheeting / Cut Bit / Bulk Clip testing:
- Wait for a recently-completed Compound Inspection batch (status=Accepted)
  to flow into the "Compound - SPP" warehouse — that's your sheeting input

> ⚠️ **Use small quantities.** Even though staging won't affect live
> production data, it WILL move staging stock around. Other testers might
> need the same batches. Be considerate.

---

## Step 5 — Have a notepad or spreadsheet ready

For each scenario you run, write down:

| Field | Where it comes from |
|---|---|
| **Reference ID** | Result modal after submit (13-digit number) |
| **Batch No** | The compound batch you tested |
| **Mix Barcode / SPP Batch No** | The compound's barcode |
| **Console record name** | e.g., `INSP-2026-00012` (parent record) |
| **Ops record name(s)** | e.g., `STE-2026-00789`, `QI-2026-00123` |
| **Legacy record name** | e.g., `MTDCR-2026-04-30-00008` |
| **Sync status** | Pending / Partial / Success / Failed |
| **Notes** | Anything weird you noticed |

A simple Google Sheet or paper notepad works. The Reference ID is the most
important — it's the glue across all 3 systems.

---

## Step 6 — Pre-flight done checklist

Tick all of these before starting any process test:

- [ ] All 4 tabs open and logged in
- [ ] Console Dash hard-refreshed (no JS console errors)
- [ ] "Console Inspection" doctype visible in Console Desk search
- [ ] Recent Legacy Bridge Log exists and shows transformed data
- [ ] Your employee IDs validated (you scanned and got green checkmarks)
- [ ] You have a test batch identified with batch_no + mix_barcode written down
- [ ] You have a notepad / spreadsheet ready to capture record names

If all 6 ticks are green, you're ready. Open the process test document
you're going to run (e.g., `01_compound_inspection.md`) and start with
its **Pre-flight checklist** section.

---

## Common environment issues & quick fixes

| Symptom | Likely cause | Quick fix |
|---|---|---|
| Dash page shows blank or "Loading…" forever | Browser cache | Hard-refresh (Cmd+Shift+R) |
| "Console Inspection" doctype not in search | Deployment hasn't completed | Wait 10 min, retry. If still missing after 30 min, file 🔴 bug |
| Employee ID scan returns "not authorized" | Employee designation not synced to Console | Ask supervisor to run employee sync |
| Bridge Log shows old "Stock Entry reference not found" errors | Pre-deployment failures | Ignore if dated before today; file bug if from after deployment |
| Sync status stuck at `Partial` for >5 minutes | Bridge callback didn't fire | File 🔴 bug — note Reference ID |
| Sync status stuck at `Pending` for >5 minutes | Bridge POST never reached the target | File 🔴 bug — note Reference ID + which leg (Ops or Legacy) |
| Two records with same Reference ID | 🚨 Critical data corruption | STOP. Call dev team immediately. |

---

## When you're done testing for the day

1. Don't leave any records in **Draft** state on Console Inspection — either
   submit them or cancel them.
2. Note in your testing log how many records you created today (helps the
   dev team understand staging stock impact).
3. Close all 4 tabs to free up Frappe Cloud session resources.

---

You're ready. Open `01_compound_inspection.md` to start the first urgent test.
