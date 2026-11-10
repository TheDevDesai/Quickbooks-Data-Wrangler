# QuickBooks Data Wrangler

A Google Sheets add-on that pulls live QuickBooks Online data into Sheets. You pick a report or data type, choose a relative date range, and the add-on writes the result to a tab. The definition is saved, and the tab updates whenever someone clicks **Sync**.

## How people use it

Everything starts from **Extensions → QuickBooks Data Wrangler** in the spreadsheet's toolbar:

- **Connect to QuickBooks** opens a short sign-in window. Each person connects once. Access is read only, and the spreadsheet never changes your books.
- **Sync now** gets the latest numbers for every synced tab and shows progress tab by tab. **Nothing updates on its own.** Numbers change only when someone clicks Sync, so a tab never shifts while someone is reading it.
- **Sync this tab only** updates just the tab you're looking at.
- **Open QuickBooks panel (large window)** opens the same panel in a floating window about twice as wide. Google fixes sidebars at about 300 pixels, so this is the way to get more room. The sidebar's **Open larger** link does the same.
- **Open QuickBooks panel** shows the side panel with:
  - the connection status
  - every synced tab with a plain status: *Synced*, *Check balances*, *Didn't sync*, or *Not synced yet*
  - when each tab was last updated
  - **Add a tab**

**Add a tab** offers one-click starting points: Profit & Loss by month, Balance Sheet, Cash roll forward, Who owes us, What we owe, and Account roll forward. Each opens its settings, including the **Accrual / Cash** switch, before anything is added. **Build a custom tab** gives access to every report, roll forward and list. The technical options are tucked under **More options**.

## What it does

- **Connect one or more QuickBooks Online companies** with Intuit's OAuth 2.0 flow. Production and sandbox are both supported.
- **Pull financial reports**:
  - Statements: Profit & Loss (summary and detail), Balance Sheet, Trial Balance, Cash Flow
  - Ledgers: General Ledger, Transaction List
  - A/R and A/P aging (summary and detail)
  - Sales by Customer, Sales by Product, Expenses by Vendor
- **Cash or accrual basis.** Every report and roll forward that supports it has an explicit **Accrual / Cash** choice. The choice is saved with the pull, so a sync never depends on the company's default setting. The basis is shown next to each synced tab in the panel and in the tab's note.
- **Report options**:
  - Columns by Month, Quarter, Year, Class, Location, Customer, and others
  - Filters by Class, Location, Customer, Vendor, and Account
  - **Custom views by GL account:** on the P&L, Balance Sheet and Trial Balance, pick accounts under **More options → Accounts**, for example a "Marketing" parent account. Sub-accounts come with their parent, section totals are recalculated to cover only what you picked, and Gross Profit and Net Income are left out. QuickBooks can't filter these reports by account, so the add-on does it after fetching. Combine it with **Show columns by Month** and a class filter for a department budget-vs-actual view.
  - Custom column lists for detail reports
- **Roll forward schedules**, by **GL account**, **bank & credit card account**, **customer (A/R)** or **vendor (A/P)**. Each row shows Beginning balance + increases − decreases = Ending balance. You can split by month, quarter or year (following the fiscal calendar) or use one period. Details are under [Roll forwards](#roll-forwards).
- **Pull raw data**: chart of accounts, customers, vendors, items, classes, invoices, bills, journal entries, payments, expenses, deposits, and more. Optionally write one row per line item, so journal entry lines or invoice lines are ready to pivot.
- **Three report layouts**:
  - **Report**: looks like QuickBooks, with indented accounts, section headers and bold totals.
  - **Table**: one row per account or transaction line with `Level 1..N` section columns and QuickBooks IDs. It has no subtotals, so `SUM`, `SUMIFS` and pivot tables are safe.
  - **Long**: an unpivoted table with one row per account per period, plus `Period Start` and `Period End` columns. Use it for multi-period P&L pivots and lookups.
- **Relative date ranges**: last month, this quarter to date, the last 12 full months, the current or prior fiscal year (from the company's fiscal year setting), or custom dates. Each sync rolls the dates forward, so "Last month" always means last month.
- **Synced tabs** are saved in the spreadsheet. Anyone connected to the same company can sync them from the menu or the panel.

## GL actuals by month

Under **Roll forwards → GL actuals by month (group, account, vendor)**, or the **GL actuals by vendor** quick pick. This is the actual amount posted to each GL account in each month, laid out like a pivot table: **GL Group → GL account → vendor** down the side, months across, with a Total column and subtotals for each account and group. It has no beginning or ending balances.

- **Basis:**
  - **Accrual** puts costs in the month they're recognized, journal entries included, so it ties to the P&L.
  - **Cash** puts them in the month they're paid.
  - **Leave out journal entries** gives a spend view: only bills, expenses, checks and card charges.
- **Vendors:** **Break out each account by vendor** is on by default. Lines with no vendor, usually journal entries, show as "(No vendor: journal entry)".
- **Accounts:** by default it covers every expense-type account (COGS, expenses, other expenses). Pick specific accounts under **More options → Accounts**.
- **GL Group:** this isn't a QuickBooks field, so each spreadsheet has a **QB Account Groups** tab mapping accounts to groups. The first sync creates it with suggested groups (COGS, Operating Expenses, Other Expenses), and later syncs add new accounts at the bottom. Edit the **Group** column to regroup. Rows can match by account ID, account number or account name, so you can paste a list of *number, name, group* without IDs. Groups appear in the order they're first listed.

### Vendors for lines with no Name (amortization, reclasses, payroll)

Prepaid subscriptions and events are paid to balance-sheet accounts, then expensed monthly by journal entries that leave **Name** blank. The vendor appears only in the line memo. GL actuals reads the memo (and the JE number) for every line and resolves a vendor in this order:

| Step | When | Vendor Source |
| --- | --- | --- |
| a | The line has a Name in QuickBooks | QBO Name |
| b | The memo contains an alias from the **Vendor Aliases** tab (whole words, any case; `/regex/` allowed) | Memo – alias |
| c | The memo matches an amortization or reclass pattern ("Amortisation for HubSpot for Jan 26", "Hubspot-Amortization for the month of Aug-26", "[Jan to Jun 2026] Google Ad Charges", "July - Google Ads transaction reclass", a vendor on the memo's first line…) and the text fuzzy-matches a QuickBooks vendor | Memo – parsed |
| d | A rule recognizes non-vendor entries: Rippling and other payroll, AES/ACS events ("Event – AES Mar-26"), deferred commissions, amortization of intangibles and depreciation, write-offs, bank charges, reimbursements, rent | Rule |
| e | A QuickBooks vendor's name appears anywhere in the memo ("To accrue … Kessler Brandt …") | Memo – parsed |
| f | Accrual / reversal or reclass with no vendor named | Rule |
| g | Parsed text that matches no vendor | Memo – parsed (not in vendor list) |
| h | Anything else | Unresolved JE / Unresolved |

- **One row per vendor:** names from aliases and memos are mapped onto the exact QuickBooks vendor name, so a vendor's bills and its amortization entries share one row.
- **Vendor Aliases tab:** created on the first sync, starting with a few example aliases (fictional vendors) that show plain, `/regex/` and account-override rows. Replace them with your own, then sync again.
- **Report vendor under GL account** (optional third column): reports *all* of that vendor's spend, named bills included, under another account. For example, a contractor can be reported under 7260 Finance Contractors and a bookkeeping firm under 7420 Audit/Accounting. The detail tab shows where each moved line is posted in QuickBooks (**Reclassed from**). The P&L tie-out still compares the posted accounts, so it keeps proving the pull matches QuickBooks.
- **Fallbacks:** a line with nothing useful in its memo falls back to its account, so bad-debt accounts become *Bad debt / write-off* and depreciation accounts become *Amortization / depreciation*.

**Outputs and controls.** Each GL actuals tab gets companion tabs:

- **`<tab> · Tie-out`:** resolved-vendor totals by account and period against QuickBooks' own Profit and Loss, with a check row that should be 0. The grouped layout also ends with a *Check vs QuickBooks P&L* row. Any difference shows as **Check balances** in the panel. The check is skipped when journal entries are left out.
- **`<tab> · Unresolved`:** every line still without a vendor, with date, account, type, JE number, amount, memo and the text found in the memo. Add aliases from here.
- **Transaction detail format:** one row per line with **Resolved Vendor**, **Vendor Source**, **Original Memo** and **JE / Doc Number**. The original **QBO Name** column is kept as is, and the tab has a filter.
- **`<tab> · Explorer`** (with the detail format): pick a GL account, then up to 10 vendors from dropdowns, or leave them blank for all. A monthly table recalculates with SUMIFS, and your picks survive each sync.
- **No double counting:** GL actuals reads expense accounts only, so a prepaid bill (debit to 1310) and its amortization (debit to 6390) can never both appear.

**Prepaid by vendor** (under Roll forwards) is the balance-sheet side: opening balance + additions − amortization = ending, per vendor and period, for asset accounts with "prepaid" in the name (or the accounts you pick). The total is checked against the Trial Balance.

## Roll forwards

In the panel, use a roll forward quick pick or **Build a custom tab → Roll forward by...**, then choose the basis:

| Roll forward by | Basis | Beginning and ending balance from | Activity from | Columns |
| --- | --- | --- | --- | --- |
| GL account | Accrual or Cash | Trial Balance on that basis | General Ledger debits and credits on that basis | Debits, Credits. Balances are debit (+) / credit (−) |
| Bank & credit card account | Accrual or Cash | Trial Balance on that basis | General Ledger, bank and credit card accounts only | Deposits/refunds (debits), payments/charges (credits) |
| Customer (A/R) | Accrual | Customer Balance report | General Ledger A/R lines, grouped by customer | Charges, payments & credits. Positive = owed to you |
| Customer | Cash | Cash received fiscal year to date | Bank and credit card lines, grouped by customer | Cash received, refunds paid |
| Vendor (A/P) | Accrual | Vendor Balance report | General Ledger A/P lines, grouped by vendor | Bills & charges, payments & credits. Positive = you owe |
| Vendor | Cash | Cash paid fiscal year to date | Bank and credit card lines, grouped by vendor | Cash paid (card charges count as paid), refunds received |

On a cash basis there are no receivables or payables. So the cash view for customers and vendors rolls forward cash received or cash paid for the fiscal year to date, resetting at each fiscal year start. There's no QuickBooks balance to compare its ending figure against.

- **Basis.** Choose Accrual or Cash. Cash-basis account roll forwards use QuickBooks' cash-basis Trial Balance and General Ledger, so A/R, A/P and similar accounts reflect cash-basis rules.
- **Ties by construction.** The first period's beginning balance comes from QuickBooks as of the day before the start date. Each later period starts from the previous period's ending balance.
- **Checked against QuickBooks.** After the roll forward is built, its final ending balances are compared with QuickBooks' own balances as of the end date. Any difference is flagged on the tab's row in the panel as **Check balances**. Common causes are edits made during the pull or a prior-period change.
- **Year-end close.** At the start of each fiscal year, income and expense accounts reset to zero and the total moves to Retained Earnings. This appears in a **Year-end Close** column, so those rows still tie.
- **Filters.** You can limit a roll forward to selected accounts, bank accounts, customers or vendors.
- **Layouts:**
  - **Schedule**: one row per account or name, with a Total row. A single period gives Beginning | Increases | Decreases | Ending. Multiple periods add a Net Change column for each period.
  - **Long**: one row per account or name per period, with Period Start, Period End, Beginning, Increases, Decreases, Net Change and Ending. Use it for pivots and lookups.
- **Zero rows.** Rows with no balance and no activity are hidden unless you check **Include rows with no balance or activity** under Advanced.

## Setup

### 1. Create an Intuit app

1. Sign in at <https://developer.intuit.com>, open **My Apps**, create an app, and select the **Accounting** scope (`com.intuit.quickbooks.accounting`).
2. Under **Keys & credentials** you'll find a **Client ID** and **Client Secret**:
   - Use the **Development** keys with a sandbox company while testing.
   - **Production** keys need the app profile and compliance questionnaire filled in. For an internal tool, the app doesn't need to be listed on the Intuit App Store.
3. You'll add a redirect URI in step 3.

### 2. Install the Apps Script project in a Google Sheet

This section installs it into a single spreadsheet, which is the quickest way to try it. To make it available in every spreadsheet at your company, install it as a private add-on instead (see [Install as a private Google Workspace add-on](#install-as-a-private-google-workspace-add-on)).

**Option A: copy and paste.** The `dist/` folder has everything as five paste-ready files.

1. Open a Google Sheet and go to **Extensions → Apps Script**. In **Project Settings**, rename the project **QuickBooks Data Wrangler**; this becomes the menu name.
2. Replace `Code.gs` with `dist/Code.gs`.
3. Add three HTML files named exactly `Sidebar`, `ConnectDialog` and `Styles`, and paste in the matching `dist/` files.
4. In **Project Settings**, check **Show "appsscript.json" manifest file in editor**, then replace the manifest with `dist/appsscript.json`.

**Option B: [clasp](https://github.com/google/clasp).**

```bash
npm install -g @google/clasp
clasp login
# Bind to an existing sheet: copy the Script ID from Apps Script → Project Settings
cp .clasp.json.example .clasp.json   # then paste your scriptId into it
clasp push
```

### 3. Configure and connect

1. Reload the sheet. **Extensions → QuickBooks Data Wrangler** appears. Choose **Open QuickBooks panel** and approve the Google permissions.
2. Open **Admin setup** at the bottom of the panel and click **Copy address** to copy the redirect address (`https://script.google.com/macros/d/<script-id>/usercallback`). Add it to your Intuit app's redirect URIs for the environment you're using.
3. Enter the Client ID and Client Secret, choose Production or Sandbox, and click **Save setup**.
4. Choose **Extensions → QuickBooks Data Wrangler → Connect to QuickBooks**, sign in, and pick the company.
5. In the panel, pick a tab under **Add a tab**, check the settings, and click **Add tab & sync**. After that, **Sync now** updates everything.

## Install as a private Google Workspace add-on

Installed this way, it appears under **Extensions** in every Google Sheet for everyone in your Google Workspace:

- The code is installed once.
- Intuit needs one redirect URI.
- The keys are entered once.
- Each person connects QuickBooks once, and it works in every spreadsheet.
- Each spreadsheet keeps its own list of synced tabs.

"Private" means only your Workspace domain can see or install it, and Google doesn't review private listings.

You need **edit access in Google Cloud** for your organization, and a **Google Workspace admin** for the last step. Google renames these console screens from time to time, so labels may differ slightly.

### 1. Create the add-on's Apps Script project
1. Go to [script.google.com](https://script.google.com) and click **New project**. This is a standalone project, not one attached to a spreadsheet. Name it **QuickBooks Data Wrangler**.
2. Paste in the five `dist/` files as in Option A above: `Code.gs`, the `Sidebar`, `ConnectDialog` and `Styles` HTML files, and `appsscript.json`.
3. In **Project Settings**, copy the **Script ID**. The add-on's Intuit redirect URI is:
   `https://script.google.com/macros/d/<Script ID>/usercallback`
   Add it to your Intuit app under **Redirect URIs**, on the Production tab, the Development tab, or both, and save.

### 2. Create a Google Cloud project for it
1. In the [Google Cloud console](https://console.cloud.google.com), create a project under your organization, e.g. "QuickBooks Data Wrangler". On its dashboard, copy the **Project number**.
2. Open **APIs & Services → OAuth consent screen** (or **Google Auth Platform**). Set **User type** to **Internal**, the app name to **QuickBooks Data Wrangler**, and add a support email.
3. Back in Apps Script, open **Project Settings → Google Cloud Platform (GCP) Project → Change project**, and paste the project number.

### 3. Test it, and enter the keys once
1. In Apps Script, choose **Deploy → Test deployments → Install** (Editor add-on). Pick a test spreadsheet and open it.
2. In that spreadsheet, choose **Extensions → QuickBooks Data Wrangler → Open QuickBooks panel → Admin setup**. Enter the Intuit Client ID and Secret, pick the environment, and list the **admins** (people allowed to change the setup).
3. Click **Save setup**. This is stored once for the whole add-on, and only admins can see or change it.
4. Connect QuickBooks and sync a tab to confirm it works.

### 4. Make a version and configure the Marketplace listing
1. In Apps Script, choose **Deploy → New deployment**, select type **Add-on**, and deploy. Note the **version number**.
2. In the Cloud console, enable the **Google Workspace Marketplace SDK**. Under **App Configuration**:
   - **App visibility:** **Private**
   - **Installation:** allow admin install (and individual install if you want)
   - **Integration:** **Sheets add-on**, with the **Script ID** and **version** from above
   - **OAuth scopes:** the same as `oauthScopes` in `appsscript.json`
   - Fill in the developer details
3. Under **Store Listing**, add the name, a short description, and the images from `marketplace/`:
   - `icon-32.png` and `icon-128.png` as the application icons
   - `card-banner-220x140.png` as the card banner
   - `screenshot-1280x800.png` as the screenshot (sample data)

   Then add your terms of service, privacy policy and support links, and click **Publish**. Private listings don't wait for Google review.

### 5. Install it for your company
In the [Google Admin console](https://admin.google.com), go to **Apps → Google Workspace Marketplace apps**. Find **QuickBooks Data Wrangler** under your internal apps and install it for the whole domain or for selected teams. Alternatively, people can install it themselves from **Extensions → Add-ons → Get add-ons**, under your company's internal apps.

### Updating the add-on later
1. Paste the new `dist/` files into the same Apps Script project.
2. Choose **Deploy → Manage deployments**, edit the add-on deployment, and select **New version**.
3. Put that version number in the Marketplace SDK's **App Configuration** and save. Everyone gets the update; synced tabs and connections are kept.

### Moving from a spreadsheet-bound copy
If you started with the code pasted into a spreadsheet:
1. Open that spreadsheet's **Extensions → Apps Script** and delete the code there, so the spreadsheet doesn't show two menus.
2. The synced tabs and their numbers stay. Open the add-on's panel and add the tabs again, using the same tab names, so the add-on keeps them in sync from then on.

## How it works

| File | Purpose |
| --- | --- |
| `src/Auth.gs` | OAuth 2.0 flow, token storage and refresh, disconnect and revoke |
| `src/Api.gs` | QuickBooks API client: retries on 429/5xx, refresh on 401, query pagination |
| `src/ReportParser.gs` | Flattens the nested Reports API JSON into the report, table and long layouts |
| `src/EntityParser.gs` | Flattens entity JSON (refs, custom fields, line items) into columns |
| `src/RollForward.gs` | Roll forward schedules: beginning balances, GL activity, year-end close, tie-out check |
| `src/Actuals.gs` | GL actuals: grouped, pivot-ready and detail layouts; P&L tie-out; Unresolved and Explorer tabs |
| `src/VendorResolver.gs` | Vendor for lines with no Name, from aliases, memo patterns, fuzzy matching and rules |
| `src/Prepaid.gs` | Prepaid roll forward by vendor |
| `src/Dates.gs` | Relative and fiscal date presets, fiscal period splitting |
| `src/Pulls.gs` | Synced tab definitions and running a sync |
| `src/Writer.gs` | Writes typed values (text, number, date formats) to a tab |
| `src/Menu.gs` | Toolbar menu (Connect, Sync now) and the server functions the panel calls |
| `src/Sidebar.html`, `src/ConnectDialog.html`, `src/Styles.html` | The QuickBooks panel, the sign-in window and their shared styles |
| `dist/` | Paste-ready build of `src/` (`npm run bundle`). Don't edit it by hand; a test fails if it's out of date |
| `marketplace/` | Icons, banner and screenshot for the private Marketplace listing |

Where things are stored:

- **Intuit app credentials** are in Script Properties. Only the admins listed in Admin setup can change them, and only they can see the setup. When installed as an add-on, other users can't open the script at all. In a spreadsheet-bound copy, anyone who can edit the Apps Script project can read them.
- **QuickBooks tokens** are in each Google user's own User Properties, keyed by company. Other editors of the sheet can't see them. Everyone who syncs needs their own connection to that company.
- **Saved pulls** are in the spreadsheet's Document Properties.

## Limits to know

- **Sync now** syncs one tab at a time, and each tab gets Apps Script's own 6-minute limit. A single very large pull, such as a multi-year General Ledger, can still hit that limit. Narrow its date range if it does.
- QuickBooks caps report size at about 400,000 cells and API calls at 500 per minute per company. For a very large General Ledger, narrow the date range or add filters.
- Each pull overwrites its destination tab. Put formulas that reference pulled data on other tabs.
- Text columns (doc numbers, IDs) are written as text so leading zeros are kept. Amounts and dates are written as real numbers and dates.

## Development

The parsers, date logic, roll forward math and pull validation are plain JavaScript. They're tested in Node against QuickBooks API JSON fixtures, and the roll forward is also tested end to end against a mocked QuickBooks API:

```bash
npm test          # parsers, roll forwards, admin rules, and a check that dist/ is current
npm run bundle    # rebuild dist/ after editing src/
```
