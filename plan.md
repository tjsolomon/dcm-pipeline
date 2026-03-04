# DCM Pipeline UI — Batch Update Plan

## Commit Groups

### Group A: Custom Domain + Channel Detection (Items 1-2) ✅
**Changes:**
1. ~~Line 6: Title tag → `Office Pipeline | DCM`~~ ✅
2. ~~Line 37: redirectUri → `https://pipeline.dcmloan.com`~~ ✅
3. ~~Lines 172-180: Add `"Channel"` to SP field select list (after `"Lender"`)~~ ✅
4. ~~Line 133 area: Add `Channel: item.Channel || ""` to `mapPipelineItem`~~ ✅
5. ~~Line 222: Change `isBkr` from `false` → `l.Channel === "Broker"`~~ ✅
6. ~~Remove the TODO comment on line 222~~ ✅

This automatically fixes VVOE N/A for Broker loans (line 262), channel labels in detail panel (line 400), and Details table Channel badge (line 515) — no additional changes needed since they already call `isBkr()`.

### Group B: Fix Overdue Loan Inclusion (Item 3) ✅
**Changes:**
~~Replace the month filter logic (lines 725-734) to include ALL active overdue loans before the selected month, not just the previous month's slipped loans:~~ ✅
```
if (fMonth !== "All") {
  const loanDate = l.DisplayClosingDate ? new Date(l.DisplayClosingDate) : null;
  const inMonth = loanDate && getMonthKey(l.DisplayClosingDate) === fMonth;
  const monthStart = getMonthStart(fMonth);
  const isOverdueActive = l.Status !== "Closed" && loanDate && loanDate < monthStart;
  if (!inMonth && !isOverdueActive) return false;
}
```

### Group C: Last Synced, My Pipeline, Clickable Stats (Items 4-6) ✅
**Item 4 — Last Synced from SP Modified:** ✅
1. ~~Add `"Modified"` to SP field select list~~ ✅
2. ~~Add `Modified: item.Modified || null` to mapPipelineItem~~ ✅
3. ~~Change line 669: compute max Modified across all loans instead of `new Date()`~~ ✅
4. ~~Change syncLabel (line 756): `"Pipeline data as of HH:MM AM/PM"`~~ ✅

**Item 5 — My Pipeline Toggle:** ✅
1. ~~Add `myPipeline` state (boolean, default false)~~ ✅
2. ~~Get logged-in user name from `msalInstance.getAllAccounts()[0]?.name`~~ ✅
3. ~~Add filter: if myPipeline, only show loans where LoanOfficer or Processor matches user~~ ✅
4. ~~Render toggle button near filter pills (filled blue when active)~~ ✅

**Item 6 — Clickable Stats Bar:** ✅
1. ~~Add `statFilter` state (null | "active" | "soon" | "urgent" | "closed")~~ ✅
2. ~~Apply stat filter as a SECOND pass after main `filtered` → produces `displayed`~~ ✅
3. ~~Stats are computed from `filtered` (unaffected by stat filter — avoids circular dependency)~~ ✅
4. ~~Views and columns use `displayed` instead of `filtered`~~ ✅
5. ~~Update `St` component to accept `onClick` and `active` props~~ ✅
6. ~~Clicking an active stat toggles it off; show clear indicator when active~~ ✅

### Group D: Escape Key, Dark Mode, Visual Polish (Items 7-9) ✅
**Item 7 — Escape Key:** ✅
~~Add `useEffect` with keydown listener for Escape → `setSel(null)`~~ ✅

**Item 8 — Dark Mode:** ✅
1. ~~Add `dark` state, initialized from `prefers-color-scheme` media query~~ ✅
2. ~~Create theme object `T` with light/dark variants for: bg, card, text, textSub, border, surface, hover, headerBg, inputBg, inputBorder~~ ✅ (implemented via CSS custom properties)
3. ~~Use `React.createContext` for theme so child components can access without prop drilling~~ ✅ (implemented via CSS custom properties instead — simpler approach)
4. ~~Toggle button (sun/moon) in header near sync label~~ ✅
5. ~~Update ALL components: App wrapper, header, stats bar, filter pills, cards, detail panel, calendar, details table, kanban columns~~ ✅
6. ~~Dark colors: bg #1b1a19, card #2d2c2b, text #e1dfdd, border #3b3a39, surface #252423~~ ✅ (updated to modern zinc palette: #111113, #1c1c1e, etc.)
7. ~~Urgency/status colors stay as-is (still visible in dark mode)~~ ✅
8. ~~MLO/Processor badge colors get dark-mode variants (more muted)~~ ✅
9. ~~Update body background via inline style on root div (CSS body style stays as fallback)~~ ✅

**Item 9 — Visual Polish:** ✅
1. ~~**Kanban cards**: Simplify — show only borrower name, closing date badge, loan amount, and ONE contextual flag (priority: conditions count > lock expiration > VVOE needed). Remove loan#/purpose/lender/MLO/Processor from card face.~~ ✅
2. ~~**Badge colors**: More pastel/muted MLO_COLORS and PROC_COLORS (used in Details view and detail panel, no longer on cards)~~ ✅
3. ~~**Header**: Add "DCM" text mark before "Office Pipeline" — small, professional~~ ✅
4. ~~**Card spacing**: Increase gap in kanban columns from 6 to 8, consistent padding~~ ✅

### Group E: Update CLAUDE.md (Item 10) ✅
~~Update to reflect:~~ ✅
- ~~Channel field now exists, broker detection working~~ ✅
- ~~Custom domain pipeline.dcmloan.com~~ ✅
- ~~My Pipeline toggle feature~~ ✅
- ~~Clickable stats bar~~ ✅
- ~~Dark mode support~~ ✅
- ~~Simplified Kanban cards~~ ✅
- ~~Last Synced shows SP Modified time~~ ✅
- ~~Escape key closes detail panel~~ ✅
