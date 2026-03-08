---
name: close-position
description: Close out a liquidity pool position. Use when exiting an LP position and recording final withdrawal amounts and prices. Updates position status, final token amounts, and final prices.
argument-hint: [protocol] [pair]
allowed-tools: mcp__google-sheets__get_sheet_data, mcp__google-sheets__update_cells
---

# Close Position

Close a liquidity pool position by updating all required fields in the Positions sheet.

## Instructions

### Step 0: Check for Raw UI Text

Before asking the user for data, check whether they have pasted raw UI text from a DeFi platform showing their final position at the time of withdrawal. Raw UI text typically contains current token amounts, prices, and balance at the time of closing.

**Detection:** If the input contains patterns like "Balance $", "Position Range", "Current Price", "Total Deposits", token amounts like "144.474911 SOL", or "WETH 9.248", treat it as raw UI text and attempt to parse it.

---

### Parsing: Orca / Turbos UI (Solana, Sui)

These values represent the final state when closing:

| Field | Pattern | Example |
|---|---|---|
| Pair | `TOKEN_A / TOKEN_B` at top | `SOL / USDC` |
| Final Token A amount | `X.xxx TOKEN_A Y.Y% $Z` | `144.474911 SOL 92.0% $11,117.98` |
| Final Token B amount | `X.xxx TOKEN_B Y.Y% $Z` | `960.365292 USDC 8.0% $960.27` |
| Final price Token A | `Current Price X.xxx TOKEN_B per TOKEN_A` | `Current Price 76.928139 USDC per SOL` |
| Final price Token B | Stablecoin (USDC/USDT) = always $1.00 | `$1.00` |
| Total balance | `Balance $X` | `Balance $12,078.25` |
| Status | `In Range` / `Out of Range` | useful context |

---

### Parsing: Uniswap / Aero UI (Ethereum, Base)

| Field | Pattern | Example |
|---|---|---|
| Pair | Pool title like `CL60-WETH/USDC` | `WETH/USDC` |
| Final Token A amount | `TOKEN_A\nX.xxx` or `TOKEN_A X.xxx` | `WETH 9.248` |
| Final Token B amount | `TOKEN_B\nX.xxx` or `TOKEN_B X.xxx` | `USDC 0` |
| Total deposit value | `Deposit $X` or `Total Deposits $X` | `$16,886.74` |
| Protocol | `Uniswap` / `Aero` labels | `Uniswap` |

**Note:** The current price of Token A (e.g., WETH) is typically not shown in the Uniswap position UI. If not present, ask the user for the final price of Token A at the time of withdrawal. USDC/USDT price is always $1.00.

---

### After Parsing

Once you've extracted what you can from the raw text:

1. **Show a summary** of the parsed values
2. **Identify missing required fields** and ask in a single follow-up:
   - Final price for Token A (if not in pasted UI text)
   - Protocol/pair identifier (if not clear from pasted text)

---

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

Assistant workflow:
1. Reads Positions sheet, finds "Turbos2 SUI/USDC" at row 25 (active)
2. Shows preview of changes
3. User confirms
4. Updates L25:M25 with prices, W25:Y25 with "Y", amounts
5. Reports success
```

```
User pastes raw Orca UI at time of withdrawal:
SOL / USDC
0.04%
Out of Range
Balance $11,950.00
Current Price 74.50 USDC per SOL
144.474911 SOL  98.0%  $10,762.00
960.365292 USDC  2.0%  $960.00
Position Range 75.9962601 — 89.0035477

Assistant parses:
- Pair: SOL/USDC
- Final SOL amount: 144.474911
- Final USDC amount: 960.365292
- Final SOL price: $74.50
- Final USDC price: $1.00

Reads Positions sheet → finds active "Orca3 SOL/USDC" at row 18.
Shows close preview with P&L calculation.
Asks for confirmation → writes update.
```
