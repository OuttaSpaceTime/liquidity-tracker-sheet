---
name: close-position
description: Close out a liquidity pool position. Use when exiting an LP position and recording final withdrawal amounts and prices. Updates position status, final token amounts, and final prices.
argument-hint: [protocol] [pair]
allowed-tools: mcp__google-sheets__get_sheet_data, mcp__google-sheets__update_cells
---

# Close Position

Close a liquidity pool position by updating all required fields in the Positions sheet.

## Instructions

When closing a position, I need the following information:
1. Protocol name and Pair (to identify the position)
2. Final amounts: TokenA and TokenB at withdrawal
3. Current prices: TokenA and TokenB prices at time of closing

## Workflow

### 1. Find the Position
- Read the Positions sheet using `mcp__google-sheets__get_sheet_data`:
  - spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
  - sheet: "Positions"
  - range: "A:Z" (to get all position data)
- Locate the row matching the protocol and pair
- Verify the position is currently active (column W = "N")
- Show the user which position will be closed (row number, entry date, entry value)

### 2. Display ASCII Preview Table
Show the user an ASCII table comparing the current values with the values that will be set after closing.

Example format:
```
Position to be closed (Row {N}): {Protocol} {Pair} on {Network}

┌──────────────────┬─────────────────┬─────────────────┐
│ Field            │ Current Value   │ New Value       │
├──────────────────┼─────────────────┼─────────────────┤
│ Closed (W)       │ N               │ Y               │
│ Curr Price A (L) │ $1.3950         │ $1.4025         │
│ Curr Price B (M) │ $1.00           │ $1.00           │
│ Final Amt A (X)  │ [empty]         │ 7384.93         │
│ Final Amt B (Y)  │ [empty]         │ 0               │
└──────────────────┴─────────────────┴─────────────────┘

Calculated values after closing:
- LP Underlying Value (Z): Will use final amounts (X*L + Y*M) = $10,352.45
- Entry Value: $10,400.00
- Realized P&L: -$47.55 (-0.46%)
```

Show the columns that will be updated:
- Columns L-M (Current prices)
- Columns W-Y (Closed status and final amounts)

### 3. Ask for Confirmation
**CRITICAL**: Use `AskUserQuestion` tool to ask the user to confirm before making any changes:
- Question: "Ready to close this position in row {N} with the values shown above?"
- Options:
  - "Yes, close it" (Recommended)
  - "No, cancel"
- If user selects "No, cancel", stop and do not proceed with the update

### 4. Update the Position (Only if Confirmed)
Use `mcp__google-sheets__update_cells` to update columns L, M, W, X, Y:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: "Positions"
- range: `L{N}:M{N}` (where N is the position row number)
- data: [[Current Price TokenA, Current Price TokenB]]

Then update the status and final amounts:
- range: `W{N}:Y{N}`
- data: [["Y", Final Amount TokenA, Final Amount TokenB]]

### 5. Report Results
After closing:
- Confirm the position is marked as closed
- Show the range that was updated
- Provide web link to view the updated sheet
- Note that LP Underlying Value (Column Z) will now calculate using final amounts

## Important Notes

- All five columns (L, M, W, X, Y) must be updated when closing a position
- Current prices should reflect the final prices at the time of withdrawal
- Final amounts should match what was actually withdrawn from the pool
- The LP Underlying Value formula automatically switches to use final amounts when Closed = "Y"

## Example Usage

```
User: "Close the SUI/USDC Turbos position with final amounts: 7384.93 SUI, 0 USDC, prices: $1.40248 SUI, $1.00 USDC"