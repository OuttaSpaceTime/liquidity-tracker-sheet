---
name: close-and-harvest
description: Harvest fees/rewards and close a liquidity pool position in a single workflow. Use when exiting an LP position where you also need to record a final harvest before closing. Combines harvest recording and position closing into one confirmed operation.
argument-hint: [protocol] [pair]
allowed-tools: mcp__google-sheets__get_sheet_data, mcp__google-sheets__update_cells
---

# Close and Harvest

Record a final harvest and close a liquidity pool position in one workflow. This combines the harvest and close-position skills to avoid two separate confirmation rounds when exiting a position.

## When to Use

Use this skill when you are fully exiting an LP position and want to record any final fees/rewards at the same time. If you only need to harvest without closing, use the `harvest` skill. If you only need to close without a harvest entry, use the `close-position` skill.

## Required Information

Collect all of the following before proceeding:

**For the harvest:**
1. Fees harvested ($)
2. Rewards harvested ($ or EUR) — 0 if none
3. How the harvest was used: compounded, withdrawn, or split (with amounts)

**For the close:**
4. Final amount TokenA (at withdrawal)
5. Final amount TokenB (at withdrawal)
6. Final price TokenA
7. Final price TokenB

## Workflow

### 1. Find the Position

Read the Positions sheet:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: "Positions"
- range: "A:Z"

- Locate the row matching the user's protocol and pair
- Verify the position is active (column W = "N")
- Note the exact Protocol identifier in column B (e.g., "Turbos2" not "Turbos") — this is critical for SUMIFS matching
- Show the user which position was found (row number, entry date, entry value, network)

### 2. Determine the Next HarvestLog Row

Read the HarvestLog to find where to append:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: "HarvestLog"
- range: "A:A"

Count existing rows to determine the next available row number.

### 3. Calculate Harvest Values

Based on how the harvest was used:

**All Compounded:** Compounded = Fees + Rewards, Remaining = 0
**All Withdrawn:** Compounded = 0, Remaining = Fees + Rewards
**Split:** Ask the user for the split. Verify Compounded + Remaining = Fees + Rewards.

### 4. Calculate Close Values

- LP Underlying Value after close = (Final Amt A × Final Price A) + (Final Amt B × Final Price B)
- Entry Value = value from column J
- Realized P&L = LP Underlying Value − Entry Value
- Realized P&L % = (P&L / Entry Value) × 100

### 5. Display Combined ASCII Preview

Show both operations in a single preview so the user can review everything at once before confirming.

Example format:
```
Position: {Protocol} {Pair} on {Network} (Row {N})

━━━ HARVEST (HarvestLog Row {H}) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────┬──────────┬────────────┬─────────────┬─────────────┬─────────────┬────────────┐
│ LP Pair (A) │ Protocol │ Date (C)   │ Fees ($) (D)│ Rewards (E) │ Compounded  │ Remaining  │
│             │ (B)      │            │             │             │ (F)         │ (G)        │
├─────────────┼──────────┼────────────┼─────────────┼─────────────┼─────────────┼────────────┤
│ SUI/USDC    │ Turbos2  │ 2/1/2026   │ $12.50      │ $0.00       │ $0.00       │ $12.50     │
└─────────────┴──────────┴────────────┴─────────────┴─────────────┴─────────────┴────────────┘

━━━ CLOSE (Positions Row {N}) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────────┬─────────────────┬─────────────────┐
│ Field            │ Current Value   │ New Value       │
├──────────────────┼─────────────────┼─────────────────┤
│ Closed (W)       │ N               │ Y               │
│ Curr Price A (L) │ $1.3950         │ $1.4025         │
│ Curr Price B (M) │ $1.00           │ $1.00           │
│ Final Amt A (X)  │ [empty]         │ 7384.93         │
│ Final Amt B (Y)  │ [empty]         │ 0               │
└──────────────────┴─────────────────┴─────────────────┘

━━━ SUMMARY ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Entry Value:          $10,400.00
  LP Value at Close:    $10,352.45
  Realized P&L:         -$47.55 (-0.46%)
  Total Fees Harvested: $12.50
  Total Withdrawn:      $12.50
```

### 6. Ask for Confirmation

**CRITICAL**: Use `AskUserQuestion` to confirm before making any changes:
- Question: "Ready to record the harvest and close this position?"
- Options:
  - "Yes, harvest and close" (Recommended)
  - "No, cancel"
- If user selects "No, cancel", stop completely. Do not write anything.

### 7. Execute Both Writes (Only if Confirmed)

Perform three updates in sequence:

**7a. Append harvest to HarvestLog:**
- sheet: "HarvestLog"
- range: `A{H}:G{H}` (H = next available row)
- data: [[LP Pair, Protocol, Date, Fees, Rewards, Compounded, Remaining]]

**7b. Update close prices on Positions:**
- sheet: "Positions"
- range: `L{N}:M{N}`
- data: [[Final Price A, Final Price B]]

**7c. Update close status and final amounts on Positions:**
- sheet: "Positions"
- range: `W{N}:Y{N}`
- data: [["Y", Final Amt A, Final Amt B]]

### 8. Report Results

Confirm both operations completed:
- Harvest recorded at HarvestLog row {H}
- Position closed at Positions row {N}
- Note that columns T, U, V on the Positions sheet will auto-update via SUMIFS
- Note that column Z (LP Underlying Value) now uses final amounts
- Provide link to the sheet: `https://docs.google.com/spreadsheets/d/1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`

## Important Notes

- **Protocol matching is critical.** The Protocol written to HarvestLog column B must exactly match column B in the Positions sheet. Read it directly from the sheet — do not guess or abbreviate.
- **Date format:** M/D/YYYY. Default to today (2/1/2026) if not specified.
- **Currency format:** Include $ for USD, EUR for euro amounts.
- **Only one confirmation prompt.** Both operations are previewed together and confirmed together. Do not ask twice.
- If the harvest amounts are both $0, skip the harvest write entirely and just close the position. Inform the user that no harvest entry was needed.

## Example Usage

```
User: "Close out Turbos SUI/USDC. Final harvest: $12.50 fees, no rewards, all withdrawn.
       Final amounts: 7384.93 SUI, 0 USDC. Prices: $1.4025 SUI, $1.00 USDC."

Assistant workflow:
1. Reads Positions sheet → finds Turbos2 SUI/USDC at row 25 (active)
2. Reads HarvestLog → next row is 14
3. Calculates: Compounded = $0, Remaining = $12.50
4. Calculates close: LP value = 7384.93 × 1.4025 + 0 × 1.00 = $10,357.46
5. Shows combined preview of harvest row 14 + close changes at row 25
6. User confirms
7. Writes HarvestLog row 14, then Positions L25:M25, then W25:Y25
8. Reports both operations complete
```
