# DCM Pipeline UI — Claude Code Project Guide

## What This Is
Single-page React application (one HTML file) that serves as the operational pipeline dashboard for Dominion Capital Mortgage (DCM). Hosted on GitHub Pages, authenticates via MSAL/Azure AD, reads and writes to SharePoint lists via REST API.

**Live URL:** https://tjsolomon.github.io/dcm-pipeline/
**Repo:** https://github.com/tjsolomon/dcm-pipeline

## Tech Stack
- **Single file:** `index.html` — contains all HTML, CSS, React JSX, and JavaScript
- **React 18** loaded via CDN (unpkg), JSX compiled by Babel standalone
- **MSAL.js 2.38** for Azure AD authentication (popup flow)
- **SharePoint REST API** for data read/write (no Graph API)
- **GitHub Pages** for hosting (auto-deploys on push to `main`)
- **No build step** — edit the HTML file directly, push, done

## Architecture

### Authentication
- Azure AD app: "DCM Pipeline UI"
- Client ID: `fda4bbf5-6010-4343-b3c9-1f82ee3b4090`
- Tenant ID: `aa14bd5b-4ade-4898-8235-d7b18e6f59e1`
- Redirect URI: `https://tjsolomon.github.io/dcm-pipeline/`
- Scope: `https://dcmloan.sharepoint.com/Sites.ReadWrite.All`
- Single tenant (DCM org only)
- Admin consent granted for Sites.ReadWrite.All (delegated)

### SharePoint Data Sources
**Site:** `https://dcmloan.sharepoint.com` (root site, no subsite)

**Office Pipeline v2 List** (~43 active loans, updated 4x/day by Power Automate)
Internal field names (verified from SharePoint API):
- Title (loan number, upsert key), LoanID, BorrowerFirstName, BorrowerLastName
- LoanOfficer, Processor, Status (Choice), PurposeType
- TotalLoanAmount (Currency), Lender (pre-merged: Investor priority, Lender fallback)
- LoanType, OccupancyType, TitleCompany
- ClosingEstimateDate, ScheduledClosingDate, ClosedDate, DisplayClosingDate (computed cascade)
- LockExpirationDate, AppraisalRequestDate, AppraisalInspectionDate
- AppraisalExpectedDeliveryDate, AppraisalDeliveryDate
- ProcessingDate, InitialSubmissionDate, ApprovedDate, ConditionSubmissionDate, ClearToCloseDate
- NewConstruction (Yes/No), OutsideTitle (Yes/No, computed), IncomeFinalized (Yes/No)
- InitialCDRequest (Date), ConditionsCount (Number, updated by conditions flow)
- Notes (Multi-line text, manual), VVOEStatus (Choice: Needed/Working/Complete, manual)

**IMPORTANT:** There is NO `Investor` field on Pipeline v2. The flow pre-merges Investor/Lender into the single `Lender` field. `OutsideTitle` is pre-computed as Yes/No. There is currently no `Channel` field (Broker vs Correspondent detection is not available — TODO).

**Conditions List** (one row per outstanding condition, updated 2x/day)
Fields: Title, LoanNumber, ConditionCategory, Condition, ResponsibleParties, ConditionCode

### Data Flow
```
SharePoint Lists ←→ Pipeline UI (this app)
       ↑
Power Automate flows (8 total) ← LendingPad exports
```
The app READS pipeline and conditions data, and WRITES Notes and VVOEStatus back to SharePoint.

## UI Structure

### Three Views
1. **Pipeline** — Kanban board, 6 status columns (Processing → Closed), sorted by closing date within columns
2. **Calendar** — Month grid (Mon-Fri only), federal holidays highlighted, loans shown on closing dates
3. **Details** — Full worksheet table with sortable columns, active loans + separate "Closed This Month" section

### Detail Panel (slide-out on card click)
- Loan info grid, LendingPad link, editable Notes (writes to SharePoint), VVOE status buttons (writes to SharePoint)
- Outstanding conditions from Conditions list, appraisal timeline, status milestones

### Visual Design
- Fluent UI / Microsoft style (Segoe UI, Microsoft color palette)
- Urgency colors: Red (≤3 days or overdue), Orange (4-7 days), Neutral gray (8+)
- No green for "normal" dates — green only used for completed states
- Status column colors: Processing=blue, Submission=purple, Approved=teal, Conditions=orange, CTC=green, Closed=gray
- MLO and Processor colored badges
- Lock expiration shown on cards with same urgency colors

### Key Business Logic
- **VVOE:** N/A for Broker loans. Auto-flags "VVOE Needed" when correspondent loan closing ≤7 days and no manual status set
- **ICD (Initial CD Request):** Flags when closing ≤7 days and InitialCDRequest is null
- **Broker detection:** Currently broken (TODO) — no Investor or Channel field on Pipeline v2. `isBkr()` returns false for all loans
- **DisplayClosingDate:** Pre-computed cascade (Closed → Scheduled → Estimate → null) by Power Automate flow
- **Lender:** Pre-merged (Investor priority, Lender fallback) by flow
- **Month filter:** UI has a month selector that filters loans by closing date. Rules:
  - Overdue active loans (past closing date, not closed) are always included regardless of selected month
  - Slipped loans from the *previous* calendar month carry into the current month view — but only once the previous month has fully passed (not mid-month)
  - Future months show only loans explicitly closing in that month
- **Processor names:** Stored and displayed in short form natively (e.g. "Chris" not "Christopher Smith") — both filter dropdowns and loan counts use the short name directly

## Known Issues / TODO

### High Priority
- [ ] **Broker/Correspondent detection** — Add Channel field to Pipeline v2 list (or Investor field), update isBkr() helper
- [ ] **Verify Conditions list field names** — Confirm LoanNumber, ConditionCategory, Condition, ResponsibleParties, ConditionCode match SharePoint internal names
- [x] **Write-back confirmed working** — Notes save and VVOE status change work with live SharePoint data (required `odata=verbose` + `__metadata` fix, see SharePoint API Notes above)

### Future Enhancements
- [ ] Status field reads as Choice object — may need `item.Status` vs `item.Status.Value` handling in REST API response
- [ ] Auto-refresh indicator (currently refreshes every 5 min silently)
- [ ] Error handling for individual field save failures
- [ ] Loading skeleton/shimmer while data fetches
- [ ] Mobile responsive improvements
- [ ] Processor-specific views (Brandy wants the Details worksheet as default)

## Development Workflow

### Making Changes
1. Edit `index.html` in VS Code
2. Test locally by opening `index.html` in browser (auth won't work locally — use mock data for UI changes, or test on GitHub Pages for API changes)
3. Commit and push to `main`
4. GitHub Pages auto-deploys in ~60 seconds
5. Test at https://tjsolomon.github.io/dcm-pipeline/ (use Incognito if auth gets stuck)

### Auth Troubleshooting
- "interaction_in_progress" error → Clear session storage in DevTools → Application → Session Storage
- Redirect goes to wrong URL → Check CONFIG.redirectUri matches Azure AD redirect URI
- 401/403 → Token expired, refresh page to re-auth

### SharePoint API Notes
- REST API base: `https://dcmloan.sharepoint.com/_api/web/lists`
- List access: `getbytitle('Office Pipeline v2')` and `getbytitle('Conditions')`
- Choice fields (Status, VVOEStatus): REST API returns the string value directly with `odata=nometadata`
- Date fields: SharePoint returns ISO format strings
- Currency fields: Returns number
- Yes/No fields: Returns boolean
- Update requires entity type lookup + IF-MATCH + X-HTTP-Method: MERGE
- **MERGE body must use `odata=verbose` Content-Type and include `__metadata: { type: entityType }` in the JSON body** — omitting either causes a 400 error. The entity type is fetched first via `?$select=ListItemEntityTypeFullName` on the list.

## File Structure
```
dcm-pipeline/
├── index.html    ← The entire application (HTML + CSS + React + API layer)
├── README.md     ← GitHub repo description
└── CLAUDE.md     ← This file (project guide for Claude Code)
```

## Related Documentation (in Claude.ai Project Knowledge)
- `dcm_project_update_v6.md` — Full project status, all 8 flows, field mappings
- `pipeline_v2_build_plan.md` — Pipeline v2 architecture, SharePoint list schemas
- `office_pipeline.md` — Legacy v1 pipeline (being sunset)
- `lock_desk_alert_synopsis.md` — Lock Desk Daily Alert flow details
- `reporting-architecture.md` — Separated architecture overview
- `kanban_mockup.html` — Original Kanban mockup
- `Mockup_2__Power_Apps_Pipeline_Board.html` — Power Apps mockup (reference only)

## Team Context
- **TJ** (project lead) — operations lead, technically proficient, prefers phased implementations
- **VP Mike** — wants clean executive dashboard (Pipeline view)
- **Chris** — processor, primary user
- **Brandy** — processor, prefers detailed worksheet view (Details view)
- **Kevin** — lending coordinator
- **MLOs** — Thomas Solomon, Amanda Reed, and others
