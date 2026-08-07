# PathReview Contribution Journal

## Week 7 — Issue Selection

**Issue chosen:** [#146 — PII scrubber fails to redact parenthesized US phone numbers](https://github.com/ascherj/pathreview/issues/146)

**Module:** `safety/pii_scrubber.py`

**Tier:** 1 (localized to a single file)

---

### What the bug is

`PIIScrubber.scrub()` and `PIIScrubber.detect()` both use a dict of compiled regex patterns. The `phone_us` pattern is:

```
\b(?:\+?1[-.]?)?\(?([0-9]{3})\)?[-.]?([0-9]{3})[-.]?([0-9]{4})\b
```

Two problems compound to make `(555) 123-4567` invisible to this pattern:

1. The leading `\b` is a word-boundary assertion. A word boundary requires a transition between a word character (`\w`) and a non-word character. In `Call me at (555) 123-4567`, the `(` sits between two non-word characters (space before, `(` itself), so `\b` fails to match there. The engine then tries matching at the first `5`, but by then the `(` is behind it and `\)?[-.]?` can no longer pick up the `)` and the space that follows it.

2. The separator class `[-.]?` only allows a dash or a dot. The `(555) 123-4567` format uses a space between `)` and `123`, so even if the area-code section matched, the rest would fail.

The same space issue also breaks the `+1 555 123 4567` international format.

---

### Why I chose this issue

- **Contained scope.** The entire fix lives in `PII_PATTERNS["phone_us"]` inside `safety/pii_scrubber.py`. No schema changes, no API changes, no cross-module coordination.

- **Pre-written failing tests.** The issue cites four tests that already exist in `tests/unit/test_pii_scrubber.py` and are currently red: `test_us_phone_number_redaction`, `test_us_phone_formats`, `test_detect_phone_pii`, `test_phone_at_start_of_text`. This is a clean red-green cycle: fix the regex, run the suite, all four should go green.

- **Safety-critical domain.** A PII scrubber that misses a common phone format is not just a unit-test failure. If a resume with a parenthesized phone number passes through `scrub()` before storage or display, that number is exposed. The safety module is where correctness matters most, which makes this a high-value fix for a small diff.

- **Checklist:**
  - Tier 1 (1-2 files)? Yes.
  - Do I understand the bug from the description alone? Yes (regex boundary + delimiter gap).
  - Are there failing tests I can run locally to verify the fix? Yes, four of them.
  - Does the fix require touching other modules? No.

---

### Plan for Week 8

The regex fix is one line. The work for next week is:

1. Run `pytest tests/unit/test_pii_scrubber.py -v` to confirm which tests are currently failing.
2. Update `phone_us` pattern to:
   - Replace leading `\b` with `(?<!\d)` (negative lookbehind for a digit) so the pattern can anchor correctly when the phone number starts with `(`.
   - Expand the separator class from `[-.]` to `[-. ]` to allow a space between groups, handling both `(555) 123-4567` and `+1 555 123 4567`.
   - Replace trailing `\b` with `(?!\d)` for symmetry.
3. Re-run the four failing tests and confirm they pass.
4. Run the full unit suite (`make test-unit`) to confirm nothing regressed.
5. Run `make check` (ruff + black + mypy) to match the project's code-style requirements before opening a PR.

---

### Files I expect to touch

| File | Change |
|---|---|
| `safety/pii_scrubber.py` | Update `phone_us` regex in `PII_PATTERNS` |
| `tests/unit/test_pii_scrubber.py` | Possibly add one additional test for a space-separated format if the existing four do not already cover it |

---

## Week 8 — Reproduction & Solution Planning

**Reproduction commit link:** https://github.com/Morgan971-pixel/ai201-pathreview/commit/edf7188

**Reproduction summary:**
Ran `pytest tests/unit/test_pii_scrubber.py -v` on the unmodified codebase and observed 5 failures: the four tests cited in the issue (`test_us_phone_number_redaction`, `test_us_phone_formats`, `test_detect_phone_pii`, `test_phone_at_start_of_text`) plus a fifth (`test_mixed_pii_and_text`) caused by a related false-positive bug in the street address pattern.

**PLAN.md link:** https://github.com/Morgan971-pixel/ai201-pathreview/blob/fix/146-pii-scrubber-parenthesized-phone-format/PLAN.md

**Walkthrough video (recommended):** N/A

**Blockers or open questions:**
None. The fix has been implemented and all 25 tests pass.

---

## Week 9 — Solution Building & PR Submission

### Check-in 1 (mid-week)

**Current progress:**
Fix implemented and all 25 unit tests passing. PLAN.md written. Public fork created at Morgan971-pixel/pathreview and branch pushed.

**Next steps:**
Open PR against ascherj/pathreview, update JOURNAL.md with PR link, submit.

**Blockers:**
None.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/482

**Branch:** fix/146-pii-scrubber-parenthesized-phone-format

**What you built:**
Fixed the phone_us regex in PIIScrubber so it correctly matches parenthesized US phone numbers like (555) 123-4567 by replacing the leading \b with a negative lookbehind and expanding the separator to include spaces. Also fixed a pre-existing false-positive bug in the street_address pattern where short suffixes like Pl were matching inside unrelated words.

**Tests added or updated:**
No new tests written — 5 pre-existing failing tests in tests/unit/test_pii_scrubber.py now pass as a result of the fix.

**Self-review confirmation:** [x] make check passes (4 pre-existing failures, 0 new)  [x] make test-unit passes (25/25)

**Draft PR feedback received from:** none

---

## Week 10 — Iteration & Reflection

### Reviewer feedback

**Feedback received:** [x] No — still awaiting review

**Summary of feedback:**
No reviewer feedback received. Per the Su26 note, reviewer feedback is not
a feature in Summer 2026.

**How you responded:**
N/A

---

### Reflection

**What was harder than you expected?**
Understanding why \b failed before ( required tracing the regex engine position
by position. \b needs a transition between a word character and a non-word
character to anchor -- but ( is non-word and the space before it is also
non-word, so no transition exists and the match never starts. That's not
obvious until you trace it manually.

**What did you learn about working in a large codebase?**
Pre-existing failures need a baseline. Before touching anything, I ran ruff
against the original file on main and found 4 errors already there. Without
that, I couldn't have honestly written "0 new failures" in the PR -- I would
have had no way to know what I caused vs what was already broken.

**How did AI tools help — and where did they fall short?**
AI was useful for explaining the regex engine behavior step by step. It fell
short when make check broke because the project's Makefile expected a .venv
that didn't exist -- no explanation helped there, I just had to read the
Makefile directly and run the tools myself.

**What would you do differently if you started over?**
Run the full test suite before reading the source code. The issue listed 4
failing tests but there were actually 5. That 5th failure exposed the
street_address bug. Running only the listed tests would have missed it.

**What are you most proud of from this module?**
Finding the street_address false positive that wasn't in the issue. The word
"applications" was being partially redacted because Pl inside it was matching
as a street suffix. One \b at the end of the pattern fixed it -- but only
because I ran the full test file instead of just the 4 listed tests.
