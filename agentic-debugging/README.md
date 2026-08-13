# Agentic Debugging & Data Repair: Bookit

Notes from two days spent working with Claude Code on a personal bookmarking app I built: [Bookit](https://github.com/nic0hh/bookit) (React Native + Expo + Supabase). This is a documented exercise in directing an AI coding agent, checking its work properly, and catching mistakes, rather than just accepting whatever it produced.

Earlier work on this app with GitHub Copilot in VS Code was a frustrating experience, Copilot kept getting stuck fixing its own mistakes in a loop. Came back to this app working with an agent in a more structured way, and paid closer attention to how I was checking its output.

## Contents

- [Setup](#setup)
- [1. Blank bookmark images bug](#1-blank-bookmark-images-bug)
- [2. Stray file caught before commit](#2-stray-file-caught-before-commit)
- [3. Pre-existing data-corruption bug](#3-pre-existing-data-corruption-bug)
- [4. AI-assisted auto-tagging feature](#4-ai-assisted-auto-tagging-feature)
- [5. Tag taxonomy cleanup](#5-tag-taxonomy-cleanup)
- [What I took from this](#what-i-took-from-this)

---

## Setup

Installed Claude Code (native Windows installer, PowerShell) and ran it directly against the Bookit repo.

---

## 1. Blank bookmark images bug

**The problem:** when the app couldn't fetch site metadata for a bookmark, it fell back to uploading a manual screenshot instead. That worked initially, but the image would eventually go blank.

Rather than diagnosing it myself and handing over a fix, I described the symptom as a user and asked the agent to investigate before touching any code. Wanted to see how it handled an open-ended problem instead of a pre-solved one.

**What it found:**
- Root cause: bookmark images live in a private storage bucket, served through signed URLs that expire after an hour. The refresh timer that's supposed to renew them only runs while the app is in the foreground. Background the app for over an hour and nothing refreshes the URL, so the image quietly breaks.
- A second, unrelated bug: if the upload itself failed, the code silently fell back to saving the phone's local temp file path as if it were the permanent image. That path gets cleared by the OS eventually, causing a second blank-image failure mode.
- A third bug found while tracing the second: on the edit-bookmark screen, a field expecting a plain file path was getting an entire object assigned to it instead.

**Where I pushed back:** the first suggested fix included extending the signed URL expiry from 1 hour to 7 days. Rejected that, it doesn't fix the actual problem, it just makes the failure less frequent. Had it implement the real fix (refresh on app resume) and skip the TTL change.

**Verification:** read the actual git diff rather than trusting the summary. Caught a real gap this way, it claimed to have added error logging for failed uploads, and the diff showed it hadn't. The three real fixes were present and correct. Confirmed the actual fix worked in production by backgrounding the app for over an hour and checking images reloaded properly, not just trusting the code review.

---

## 2. Stray file caught before commit

Before committing the fix above, `git status` showed an unfamiliar file about to be pushed to a public repo, with "token" and "supabase" in the name. Didn't assume it was safe, didn't panic and delete it either.

The filename had a hidden character that made it hard to reference directly in PowerShell, worked around it by selecting the file as an object rather than typing the name. Checked the contents for anything resembling a real key or credential once I could actually read it. Turned out to be an old local search-results dump from months earlier, nothing sensitive. Separately confirmed `.env` (the file that actually holds real credentials) was properly gitignored, so nothing was at risk either way.

Deleted the stray file, pushed the real fix.

---

## 3. Pre-existing data-corruption bug

While building the tagging feature below, the agent flagged an issue in the shared `updateBookmark` function. It always sent `notes` and `folder_id` on every update, even when those fields weren't meant to change, using a fallback that defaults to `null` instead of omitting the field. In JavaScript, an omitted field gets dropped from the request; a field explicitly set to `null` still gets sent and overwrites. So any partial update, including the app's "fix image dimensions" maintenance action, was silently wiping notes and folder assignments on every bookmark it touched.

Asked directly whether this was confirmed or a theoretical read of the code. It said plainly it had traced the bug from source but had no database access, so it didn't know if it had actually fired. Asked for a read-only diagnostic query instead of more speculation and ran it myself in the Supabase SQL editor. 68 bookmarks came back affected. Reviewed the list manually and didn't find any notes I remembered writing that were missing, so the practical damage looked minimal.

For the repair: preview query first to see the exact change before anything ran, then the actual update wrapped in a transaction, checked the result was 0 remaining before committing. No write went through unchecked.

---

## 4. AI-assisted auto-tagging feature

**The problem:** bookmarks saved on the go rarely got tagged, leaving a backlog that was hard to search.

**Build decisions:**
- Auto-apply tags directly, no approval step, since tags are cheap to fix individually if wrong. Left notes generation out entirely, didn't want AI silently writing into a field that's meant to be my own voice.
- Claude Haiku 4.5 via the API with structured output for reliable tag arrays. Checked actual cost against real usage (~10 saves/month), came out to roughly a cent a month.
- Fed it my existing tags as a hint to reduce near-duplicate creation, plus the folder name so hobby-specific bookmarks (Ravelry knitting patterns) surfaced specific tags instead of generic ones.

**Before wiring it in**, tested against three real bookmarks and checked whether specific terms like "intarsia" or "fairisle" were genuinely present on the source page, not just plausible output from the model's general knowledge of a designer's style. Pulled the raw scraped content myself to confirm. One of the three came back more generic, traced to Ravelry gating full pattern detail behind login for unauthenticated scrapers.

**After running it on the full backlog** (313 bookmarks), fixed three real gaps: no loading indicator during the run, no way to identify failed bookmarks, no automatic refresh after completion. In the process, also found and fixed an unrelated bug where the bookmark list only merged in new items instead of picking up field updates on existing ones, which is why tags weren't showing without a manual reload.

---

## 5. Tag taxonomy cleanup

Running the tagging feature surfaced real inconsistency: colorwork vs. colourwork, hair and jewellery items tagged unevenly or not at all.

Pulled the full tag frequency table (940 unique tags across 313 bookmarks) plus a similarity scan to catch likely duplicates, rather than working off the handful of examples I'd spotted myself. Reviewed the similarity list by hand rather than trusting the scores, a lot of it was coincidental (sheer/shelter, chair matching as a substring of hair).

Caught one before it became a problem: dk (a yarn weight) and doubleknit (an unrelated knitting technique) would likely have been flagged as similar and merged, since they share letters. Flagged this explicitly before any merge ran, domain knowledge a similarity score has no way to know.

Also found several tags that were scraping artifacts, internal category slugs from DROPS Design's site metadata that had leaked into the tag column. Deleted those rather than merging.

Accessory/hair category boundaries were a judgment call made by hand: footwear excluded, socks/legwarmers/shawls included, ponchos and wrap-style garments excluded despite lexical overlap.

Reading the actual preview query output rather than the plan caught two real bugs: "hair" was triggering the accessory backfill on hair-care products (masks, mousse) purely from sharing that one tag, and "wrap" was ambiguous between real accessory wraps and wrap-style cardigans in the actual data. Fixed both, re-ran the preview to confirm, then the transaction-wrapped write, same discipline as the data repair.

---

## 6. Regression from an earlier fix

**The problem:** after shipping the blank-image fix above, saving a new bookmark stopped working entirely. Pressing Save did nothing, no error, no feedback.

**Investigation:** told the agent explicitly to look through the code first and report back what it found before proceeding with anything. It attempted to install a browser automation extension to reproduce the bug interactively, without checking in first, despite the direct instruction. Called this out directly. It gave a straightforward answer rather than defending the choice: static analysis alone (reading the save handler, grepping for related calls, checking the installed react-native-web package source, and a couple of curl checks on image CORS headers) would have been enough, no browser needed, and it had jumped ahead of what was asked.

**What it found:** the blank-image fix itself had introduced the regression. It had changed a silent fallback into a hard failure, reasonable in isolation, but that hard failure now fired on nearly every save, because the app tries to re-upload the bookmark's preview image, and that upload fails for any site (Ravelry included) that doesn't allow cross-origin image fetches. Worse, the error meant to explain the failure was built on a function that's a documented no-op on the web build, so the failure was completely silent. Three separate issues stacking: a reasonable fix, an unrelated site-side restriction, and a pre-existing dead alert system, not one bad decision.

**The fix:** hotlink remote preview images directly instead of re-fetching and re-uploading them (sidesteps the CORS failure for the common case), and replace the silent alert calls with a real cross-platform one, applied consistently across every screen using the same pattern, not just the one that broke.

**Verification:** confirmed the dev bundle rebuilt clean, then tested for real rather than trusting that. A non-Ravelry bookmark saved correctly with its image. A Ravelry pattern URL saved correctly too, confirming the fix held.

---

## What I took from this

- Checking the agent's actual output against real evidence, a diff or a query result, rather than trusting its summary. Caught a gap between claimed and actual more than once.
- Making real calls with incomplete information: rejecting a plausible-sounding but wrong fix (the TTL change), and catching a plausible-sounding but wrong assumption (dk/doubleknit) that needed outside domain knowledge to catch, not just careful reading.
- Handling the stray-file moment properly: no assumption of safety, no overreaction either, just verification before a decision.
- Never running a write to real data without a preview and a checked result first.
- Describing problems as experienced rather than handing over a pre-made diagnosis, and letting the agent investigate from there.
- Catching an agent acting past an explicit instruction, not just a vague process preference. I told it directly to look through the code and report back before doing anything. It attempted to install a browser extension anyway without checking in. Naming that clearly got a real correction rather than a justification.
