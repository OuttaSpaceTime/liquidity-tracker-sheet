---
name: add-position
description: Add a new liquidity pool position to the tracker. Use when creating a new LP position entry across Orca, Aero, Turbos, or other protocols. Automatically assigns protocol numbers, calculates entry values, and creates tracking formulas.
argument-hint: [protocol] [pair]
allowed-tools: mcp__google-sheets__get_sheet_data, mcp__google-sheets__update_cells
---

# Add Position

Add a new liquidity pool position to the Positions sheet with all formulas properly configured.

## Instructions

### Step 0: Check for Raw UI Text

Before asking for data, check whether the user has pasted raw UI text from a DeFi platform (Orca, Turbos, Uniswap, Aero, etc.). Raw UI text is unstructured copy-pasted output from the browser — it typically contains token amounts, prices, ranges, and balances mixed with labels.

**Detection:** If the input contains patterns like "Balance $", "Pending Yield", "Position Range", "Current Price", "Total Deposits", "Swap Fees", "Deposit", "In Range", "Out of Range", or token amount lines like "144.474911 SOL", treat it as raw UI text and attempt to parse it.

---

### Parsing: Orca / Turbos UI (Solana, Sui)

Orca and Turbos share a similar UI layout. Look for these patterns:

| Field | Pattern | Example |
|---|---|---|
| Pair | `TOKEN_A / TOKEN_B` at top | `SOL / USDC` |
| Fee tier | Standalone `0.04%` or `0.3%` near the pair | `0.04%` |
| Status | `In Range` or `Out of Range` | `In Range` |
| Total balance | `Balance $X` | `Balance $12,078.25` |
| Current price | `Current Price X.xxx TOKEN_B per TOKEN_A` | `Current Price 76.928139 USDC per SOL` |
| Position range | `Position Range X.xxx — Y.xxx` | `Position Range 75.9962601 — 89.0035477` |
| Token A amount | Line with `X.xxx TOKEN_A` followed by `%` and `$` | `144.474911 SOL 92.0% $11,117.98` |
| Token B amount | Line with `X.xxx TOKEN_B` followed by `%` and `$` | `960.365292 USDC 8.0% $960.27` |
| Pending yield / accrued fees | `Pending Yield $X` | `Pending Yield $70.84` |
| 24h yield | `24H X.xxx% $X` | `24H 0.531% $64.13` |

**Inferred fields:**
- Protocol: "Orca" if the UI mentions "Orca" or looks like the Orca interface; "Turbos" if it mentions "Turbos"
- Network: "Solana" for Orca; "Sui" for Turbos
- Current price of stablecoin (USDC): always $1.00
- Entry prices = current prices (for a new position just opened)
- Entry amounts = current amounts (for a new position just opened)

---

### Parsing: Uniswap / Aero UI (Ethereum, Base)

Uniswap and Aero (Aerodrome on Base) use a similar interface layout. Look for:

| Field | Pattern | Example |
|---|---|---|
| Pair | In pool title like `CL60-WETH/USDC` or `WETH/USDC` | `CL60-WETH/USDC` |
| Fee tier | Standalone `0.3%` near pool name | `0.3%` |
| Protocol | `Uniswap logo` / `Uniswap` label; `Aero` / `Aerodrome` label | `Uniswap` |
| Total deposit | `Total Deposits $X` or `Deposit $X` | `Total Deposits $16,886.74` |
| Token A amount | `TOKEN_A\nX.xxx` or `TOKEN_A X.xxx` | `WETH 9.248` |
| Token B amount | `TOKEN_B\nX.xxx` or `TOKEN_B X.xxx` | `USDC 0` |
| Earned (total) | `Earned $X` | `Earned $91.696` |
| Accrued swap fees | `Swap Fees X.xxx TOKEN_A X.xxx TOKEN_B` | `Swap Fees 0.026 WETH 43.766 USDC` |
| APR | `APR X%` or `(A/W/D)PR X%` | `10.2%` |

**Inferred fields:**
- Protocol: "Uniswap" if "Uniswap" appears; "Aero" or "Aerodrome" if those appear
- Network: "Ethereum" for Uniswap; "Base" for Aero
- Current price of stablecoin (USDC/USDT): always $1.00
- Entry value = Total Deposits value
- Entry amounts = current token amounts (for a freshly opened position)
- For WETH price: not always in the UI — may need to ask the user

---

### After Parsing

Once you've extracted what you can from the raw text:

1. **Show a summary** of the parsed values in a clear list
2. **Identify missing required fields** and ask the user for them in a single follow-up question. Required fields you cannot infer:
   - Protocol (if not identifiable from pasted text)
   - Network (if not identifiable)
   - Entry price for Token A (if not a stablecoin and not in pasted text)
   - Entry price for Token B (if not a stablecoin)
   - Fees paid to enter the position (transaction fees — ask if not specified, default $0)
   - Date entered (default to today: 2/24/2026)

---

## Workflow

### 1. Collect Information

Gather all required data — either parsed from raw UI text (Step 0) or asked directly.

**Required Data:**
1. **Date Entered** (default to today if not specified)
2. **Protocol** (Orca, Aero, Turbos, Uniswap, etc.)
3. **Pair** (SOL/USDC, ETH/USDC, SUI/USDC, WETH/USDC, etc.)
4. **Network** (Solana, Base, Sui, Ethereum)
5. **Price Range** (format: "low — high")
6. **Entry Price TokenA** (in USD)
7. **Entry Price TokenB** (usually $1.00 for USDC/USDT)
8. **Amount TokenA (entry)**
9. **Amount TokenB (entry)**

**Optional Data:**
- **Fees Paid** (transaction fees to enter position, default $0)
- **Current amounts** (if different from entry, otherwise copy entry amounts)
- **Accrued Fees** (from pending yield, default to $0)
- **Reward token info** (symbol, amount, price)

### 2. Determine Unique Protocol Identifier

**CRITICAL**: Each position must have a unique Protocol identifier for SUMIFS formulas to work correctly.

- Read all existing positions using `mcp__google-sheets__get_sheet_data`:
  - spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
  - sheet: "Positions"
  - range: "B:B" (Protocol column)
- Find all protocols that start with the base protocol name (e.g., "Turbos", "Turbos2", "Turbos3")
- Extract the highest number suffix:
  - If only "Turbos" exists, the highest number is 1 (implied)
  - If "Turbos", "Turbos2", "Turbos3" exist, the highest is 3
- Set the new protocol identifier to: `{Protocol}{N+1}` where N is the highest existing number
  - If this is the first position of this protocol, use just the protocol name (e.g., "Turbos")
  - If other positions exist, append the next number (e.g., "Turbos2", "Turbos3")

**Examples:**
- First Turbos position: "Turbos"
- Second Turbos position: "Turbos2"
- Third Turbos position: "Turbos3"
- First Orca position: "Orca"
- Eleventh Orca position: "Orca11"

### 3. Calculate Values
- **Total Entry Value ($)**: (Amount TokenA × Price TokenA) + (Amount TokenB × Price TokenB)
- **Current Price TokenA/B**: Default to entry prices unless specified otherwise
- **Current Amount TokenA/B**: Default to entry amounts unless specified otherwise

### 4. Determine Next Row Number
- Read the Positions sheet using `mcp__google-sheets__get_sheet_data`:
  - spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
  - sheet: "Positions"
  - range: "A:A" (to count rows)
- Find the last row with data
- Next row number = last row + 1
- This row number (N) will be used in formulas

### 5. Build Row Data with Formulas

Create a row with all 26 columns (A-Z):

**Columns A-K (Entry Data):**
- A: Date Entered
- B: Protocol
- C: Pair
- D: Network
- E: Range (low-high)
- F: Entry Price TokenA ($)
- G: Entry Price TokenB ($)
- H: Amount TokenA (entry)
- I: Amount TokenB (entry)
- J: Total Entry Value ($)
- K: Fees Paid ($)

**Columns L-S (Current Position Data):**
- L: Current Price TokenA ($)
- M: Current Price TokenB ($)
- N: Amount TokenA (current)
- O: Amount TokenB (current)
- P: Accrued Fees (unharvested $)
- Q: Reward Token (sym)
- R: Rewards Amount (unharvested)
- S: Reward Token Price (EUR)

**Columns T-V (Harvested Totals - FORMULAS):**
- T: `=SUMIFS(HarvestLog!D:D,HarvestLog!B:B,BN,HarvestLog!A:A,CN)` (Total Harvested Fees)
- U: `=SUMIFS(HarvestLog!E:E,HarvestLog!B:B,BN,HarvestLog!A:A,CN)` (Total Harvested Rewards)
- V: `=SUMIFS(HarvestLog!$F:$F,HarvestLog!$B:$B,BN,HarvestLog!$A:$A,CN)` (Total Compounded)

**Columns W-Y (Status & Exit Data):**
- W: "N" (Closed - new positions are active)
- X: "" (Final Amount TokenA - empty for active positions)
- Y: "" (Final Amount TokenB - empty for active positions)

**Column Z (Calculated Value - FORMULA):**
- Z: `=IF(WN="Y",(XN*LN)+(YN*MN),(NN*LN)+(ON*MN))` (LP Underlying Value)

Where N is the row number.

### 6. Display ASCII Preview Table

Create an ASCII table showing the row that will be added. Include:
- Row number where data will be inserted
- The unique protocol identifier (e.g., "Turbos2")
- All column values (A-Z)
- Formulas shown with actual row numbers

Example format:
```
┌──────────────┬────────────┬────────────┬─────────┬────────────┬─────────┬─────────┬─────────┬─────────┬────────┬────────┐
│ Date         │ Protocol   │ Pair       │ Network │ Range      │ Entry $ │ Entry $ │ Amt A   │ Amt B   │ Total  │ Fees   │
│ Entered (A)  │ (B)        │ (C)        │ (D)     │ (E)        │ A (F)   │ B (G)   │ (H)     │ (I)     │ $ (J)  │ $ (K)  │
├──────────────┼────────────┼────────────┼─────────┼────────────┼─────────┼─────────┼─────────┼─────────┼────────┼────────┤
│ 1/26/2026    │ Turbos2    │ SUI/USDC   │ Sui     │ 1.39-1.70  │ $1.4002 │ $1.00   │ 7430.76 │ 0       │ $10400 │ $5.25  │
└──────────────┴────────────┴────────────┴─────────┴────────────┴─────────┴─────────┴─────────┴─────────┴────────┴────────┘

┌─────────┬─────────┬──────────┬──────────┬────────────┬────────┬────────┬──────────┐
│ Curr $  │ Curr $  │ Curr Amt │ Curr Amt │ Accrued    │ Reward │ Reward │ Reward $ │
│ A (L)   │ B (M)   │ A (N)    │ B (O)    │ Fees $ (P) │ Sym(Q) │ Amt(R) │ (S)      │
├─────────┼─────────┼──────────┼──────────┼────────────┼────────┼────────┼──────────┤
│ $1.4002 │ $1.00   │ 7430.76  │ 0        │ $0         │        │        │          │
└─────────┴─────────┴──────────┴──────────┴────────────┴────────┴────────┴──────────┘

┌──────────────────────────────────────┬──────────────────────────────────────┬──────────────────────────────────────┐
│ Total Harvested Fees $ (T)           │ Total Harvested Rewards $ (U)        │ Total Compounded $ (V)               │
├──────────────────────────────────────┼──────────────────────────────────────┼──────────────────────────────────────┤
│ =SUMIFS(HarvestLog!D:D,...)          │ =SUMIFS(HarvestLog!E:E,...)          │ =SUMIFS(HarvestLog!$F:$F,...)        │
└──────────────────────────────────────┴──────────────────────────────────────┴──────────────────────────────────────┘

┌────────┬──────────────┬──────────────┬─────────────────────────────────────────┐
│ Closed │ Final Amt A  │ Final Amt B  │ LP Underlying Value $ (Z)               │
│ (W)    │ (X)          │ (Y)          │                                         │
├────────┼──────────────┼──────────────┼─────────────────────────────────────────┤
│ N      │              │              │ =IF(WN="Y",(XN*LN)+(YN*MN),(NN*LN)+...) │
└────────┴──────────────┴──────────────┴─────────────────────────────────────────┘
```

### 7. Ask for Confirmation

**CRITICAL**: Use `AskUserQuestion` tool to ask the user to confirm before making any changes:
- Question: "Ready to add this position to row {N} of the Positions sheet?"
- Options:
  - "Yes, add it" (Recommended)
  - "No, cancel"
- If user selects "No, cancel", stop and do not proceed with the update

### 8. Append the Row (Only if Confirmed)

Use `mcp__google-sheets__update_cells` to append the row:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: `Positions`
- range: `A{N}:Z{N}` (where N is the next row number determined in step 4)
- data: [[row data with formulas as strings]]

Note: Formulas must be entered as strings (e.g., "=SUMIFS(HarvestLog!D:D,...)") and will be interpreted by Google Sheets

### 9. Report Results

After adding:
- Confirm which row the position was added to
- Show the actual updated range
- Provide web link to view the updated sheet
- Confirm that formulas are working (show calculated values from columns T, U, V, Z)

## Important Notes

### Protocol Numbering (CRITICAL)
- **Every position MUST have a unique Protocol identifier**
- The SUMIFS formulas match on both Protocol (column B) AND Pair (column C)
- If two positions have the same Protocol name, their harvest totals will be incorrectly combined
- Always increment the protocol number for each new position:
  - "Turbos" → "Turbos2" → "Turbos3"
  - "Orca" → "Orca2" → "Orca3" ... → "Orca10"
- When closing a position, the HarvestLog entries must match the exact Protocol identifier
- This ensures each position's harvested fees/rewards are tracked separately

### Formula Row Numbers
- **CRITICAL**: The row number in formulas must match the actual row where data is appended
- Always read the sheet first to determine the next row number
- Account for the fact that append might skip blank rows

### Formula Patterns
The SUMIFS formulas pull data from HarvestLog sheet:
- Match Protocol (column B) and Pair (column C) with harvest records
- Sum fees, rewards, or compounded amounts accordingly

The IF formula for LP Underlying Value:
- Uses final amounts (X, Y) if position is closed (W="Y")
- Uses current amounts (N, O) if position is active (W="N")

### Value Formatting
- Currency values: Include $ or EUR symbol
- Dates: Use M/D/YYYY format
- Ranges: Use em dash (—) between low and high prices
- Numbers: Use proper decimal places (no excessive precision)

## Example Usage

```
User: "Add new position: Turbos SUI/USDC on Sui, 7430.76 SUI, range 1.39526-1.70075, entry price $1.40021"

Assistant response:
- Collects any missing data (date, token B amount, etc.)
- Calculates total entry value
- Shows preview of all data and formulas
- Appends to Positions sheet with formulas for row N
- Confirms success and shows results
```

```
User pastes raw Orca UI text:
SOL / USDC
0.04%
In Range
Balance $12,078.25
Current Price 76.928139 USDC per SOL
75.996  89.004
Position Range 75.9962601 — 89.0035477
144.474911 SOL  92.0%  $11,117.98
960.365292 USDC  8.0%  $960.27
Pending Yield $70.84

Assistant parses:
- Pair: SOL/USDC
- Protocol: Orca (inferred)
- Network: Solana (inferred)
- Range: 75.9962601 — 89.0035477
- Current Price SOL: $76.928139
- SOL amount: 144.474911
- USDC amount: 960.365292
- Balance: $12,078.25
- Accrued fees: $70.84

Then asks: "Parsed the following from your Orca UI. Is this a new position? If so, confirm entry price for SOL was also $76.928139, and provide fees paid to enter."
Shows preview → confirms → writes row.
```
