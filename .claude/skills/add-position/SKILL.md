---
name: add-position
description: Add a new liquidity pool position to the tracker. Use when creating a new LP position entry across Orca, Aero, Turbos, or other protocols. Automatically assigns protocol numbers, calculates entry values, and creates tracking formulas.
argument-hint: [protocol] [pair]
allowed-tools: mcp__google-sheets__get_sheet_data, mcp__google-sheets__update_cells
---

# Add Position

Add a new liquidity pool position to the Positions sheet with all formulas properly configured.

## Instructions

When adding a new position, I need the following information:

### Required Data
1. **Date Entered** (default to today if not specified)
2. **Protocol** (Orca, Aero, Turbos, etc.)
3. **Pair** (SOL/USDC, ETH/USDC, SUI/USDC, etc.)
4. **Network** (Solana, Base, Sui)
5. **Price Range** (format: "low — high")
6. **Entry Price TokenA** (in USD)
7. **Entry Price TokenB** (usually $1.00 for USDC)
8. **Amount TokenA (entry)**
9. **Amount TokenB (entry)**

### Optional Data
- **Fees Paid** (transaction fees to enter position)
- **Current amounts** (if different from entry, otherwise copy entry amounts)
- **Accrued Fees** (default to $0)
- **Reward token info** (symbol, amount, price)

## Workflow

### 1. Collect Information
Gather all required data from the user. If any required field is missing, ask for it.

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

### 6. Preview Data
Show the user:
- The unique protocol identifier that will be used (e.g., "Turbos2")
- All data values that will be entered
- The formulas that will be created (with actual row numbers)
- Calculated total entry value

### 7. Append the Row
Use `mcp__google-sheets__update_cells` to append the row:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: `Positions`
- range: `A{N}:Z{N}` (where N is the next row number determined in step 4)
- data: [[row data with formulas as strings]]

Note: Formulas must be entered as strings (e.g., "=SUMIFS(HarvestLog!D:D,...)") and will be interpreted by Google Sheets

### 8. Report Results
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
