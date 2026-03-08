---
name: harvest
description: Record harvested fees and rewards from a liquidity pool position. Use when claiming fees or rewards from LP positions, tracking whether amounts were compounded (reinvested) or withdrawn. Links harvest data to positions via protocol identifier.
argument-hint: [protocol] [pair]
allowed-tools: mcp__google-sheets__get_sheet_data, mcp__google-sheets__update_cells
---

# Harvest

Record harvested fees and rewards to the HarvestLog sheet for a liquidity pool position.

## Instructions

### Step 0: Check for Raw UI Text

Before asking the user for data, check whether they have pasted raw UI text from a DeFi platform. Raw UI text typically contains token amounts, pending yields, and balance information mixed with labels.

**Detection:** If the input contains patterns like "Pending Yield", "Earned", "Swap Fees", "Harvest Yield", "Fees Harvested", or token-denominated fee amounts like "0.026 WETH", treat it as raw UI text and attempt to parse it.

---

### Parsing: Orca / Turbos UI (Solana, Sui)

| Field | Pattern | Example |
|---|---|---|
| Pair | `TOKEN_A / TOKEN_B` at top | `SOL / USDC` |
| Pending yield (total fees) | `Pending Yield $X` | `Pending Yield $70.84` |
| Harvest yield label | `Harvest Yield` (appears after "Pending Yield $X") | confirms harvesting intent |
| 24h yield | `24H X.xxx% $X` | `24H 0.531% $64.13` |

From Orca/Turbos UI:
- **Fees Harvested ($)** = value from `Pending Yield $X`
- **Rewards Harvested ($)** = 0 unless a separate reward token line is present
- Protocol/Pair: inferred from pair at top of UI

---

### Parsing: Uniswap / Aero UI (Ethereum, Base)

| Field | Pattern | Example |
|---|---|---|
| Pair | Pool title like `CL60-WETH/USDC` | `WETH/USDC` |
| Earned total | `Earned $X` | `Earned $91.696` |
| Accrued swap fees | `Swap Fees X.xxx TOKEN_A X.xxx TOKEN_B` | `Swap Fees 0.026 WETH 43.766 USDC` |
| Protocol | `Uniswap` / `Aero` labels | `Uniswap` |

From Uniswap/Aero UI:
- **Fees Harvested ($)** = value from `Earned $X` or the dollar-converted swap fees
- If swap fees are shown per-token (e.g., `0.026 WETH 43.766 USDC`), you need the token price to compute USD value — ask if not known
- **Rewards Harvested ($)** = 0 for standard Uniswap V3 (no rewards unless it's a farm)

---

### After Parsing

Once you've extracted what you can from the raw text:

1. **Show a summary** of the parsed values
2. **Identify missing required fields** and ask in a single follow-up:
   - How the harvest was used: compounded (reinvested), withdrawn, or split?
   - Exact dollar amounts if only token quantities were in the UI and price is needed

---

## Workflow

### 1. Identify the Position

**CRITICAL**: The Protocol identifier must match EXACTLY with the Positions sheet.

- Read the Positions sheet using `mcp__google-sheets__get_sheet_data`:
  - spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
  - sheet: "Positions"
  - range: "A:W" (to get all position data including Closed status)
- Find active positions (column W = "N")
- Match the Pair the user mentioned (column C)
- Show the user the exact Protocol identifier to confirm (e.g., "Turbos2" not just "Turbos")
- This ensures the SUMIFS formulas will correctly link harvests to positions

**Example:**
```
User says: "Harvest from SUI/USDC Turbos"
Assistant reads Positions sheet and finds:
- Row 22: Turbos, SUI/USDC (Closed)
- Row 25: Turbos2, SUI/USDC (Active)
Assistant confirms: "Found active Turbos2 SUI/USDC position. Recording harvest for Turbos2."
```

### 2. Collect Harvest Data

Gather:
- Fees Harvested amount (in $)
- Rewards Harvested amount (in $ or EUR)
- Total harvest value for confirmation
- How the harvest was used (all compounded / all withdrawn / split)

### 3. Calculate Compounded and Remaining

Based on how the harvest was used:

**All Compounded (reinvested):**
- Compounded = Total harvest value
- Remaining = 0

**All Withdrawn:**
- Compounded = 0
- Remaining = Total harvest value

**Split (some reinvested, some withdrawn):**
- Ask user for the split amounts
- Compounded = amount reinvested
- Remaining = amount withdrawn
- Verify: Compounded + Remaining = Total harvest value

### 4. Display ASCII Preview Table

Show the user an ASCII table of what will be added to HarvestLog. Include the row number where the entry will be inserted.

Example format:
```
Row {N} will be added to HarvestLog:

┌─────────────┬──────────┬────────────┬─────────────────────┬───────────────────────┬─────────────┬────────────┐
│ LP Pair (A) │ Protocol │ Date (C)   │ Fees Harvested ($)  │ Rewards Harvested ($) │ Compounded  │ Remaining  │
│             │ (B)      │            │ (D)                 │ (E)                   │ (F)         │ (G)        │
├─────────────┼──────────┼────────────┼─────────────────────┼───────────────────────┼─────────────┼────────────┤
│ SUI/USDC    │ Turbos2  │ 1/26/2026  │ $47.75              │ $20.68                │ $68.43      │ $0.00      │
└─────────────┴──────────┴────────────┴─────────────────────┴───────────────────────┴─────────────┴────────────┘
```

Display a summary:
- Total harvest value: [sum of fees + rewards]
- Compounded (reinvested): [amount]
- Withdrawn: [remaining amount]
- This will update Position sheet totals automatically via SUMIFS formulas

### 5. Ask for Confirmation

**CRITICAL**: Use `AskUserQuestion` tool to ask the user to confirm before making any changes:
- Question: "Ready to add this harvest entry to row {N} of the HarvestLog sheet?"
- Options:
  - "Yes, add it" (Recommended)
  - "No, cancel"
- If user selects "No, cancel", stop and do not proceed with the update

### 6. Append to HarvestLog (Only if Confirmed)

First, determine the next row in HarvestLog using `mcp__google-sheets__get_sheet_data`:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: "HarvestLog"
- range: "A:A" (to count rows)

Then use `mcp__google-sheets__update_cells` to append the row:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: `HarvestLog`
- range: `A{N}:G{N}` (where N is the next row number)
- data: [[LP Pair, Protocol, Date, Fees ($), Rewards ($), Compounded, Remaining]]

### 7. Report Results

After adding:
- Confirm the harvest was recorded
- Show which row it was added to
- Note that the Positions sheet formulas will automatically update
- Provide web link to view the HarvestLog

## Important Notes

### Protocol Identifier Matching (CRITICAL)
- **The Protocol in HarvestLog MUST exactly match the Protocol in Positions sheet**
- The SUMIFS formulas in Positions columns T, U, V match on:
  - Protocol (column B in both sheets)
  - Pair (column A in HarvestLog, column C in Positions)
- If the protocol doesn't match exactly, the harvest won't be counted
- Examples of correct matches:
  - Position: "Turbos2" → HarvestLog: "Turbos2" ✓
  - Position: "Orca10" → HarvestLog: "Orca10" ✓
  - Position: "Turbos2" → HarvestLog: "Turbos" ✗ (won't match!)

### Currency Formatting
- Include $ symbol for USD amounts
- Include EUR symbol if rewards are in EUR
- Use format like: "$47.75", "€20.68"

### Date Formatting
- Use M/D/YYYY format (e.g., "12/27/2025")
- Default to today's date if not specified

### Compounded vs Remaining
- **Compounded**: Amount reinvested back into the liquidity pool
  - Updates the "Total Compounded" column in Positions (column V)
  - Should increase the position's entry value for accurate ROI calculations
- **Remaining**: Amount withdrawn from the protocol
  - Represents realized profit taken out
  - Not reinvested in the position

### Linking to Positions
The Positions sheet has these formulas that pull from HarvestLog:
- Column T: `=SUMIFS(HarvestLog!D:D,HarvestLog!B:B,B[row],HarvestLog!A:A,C[row])` (Total Fees)
- Column U: `=SUMIFS(HarvestLog!E:E,HarvestLog!B:B,B[row],HarvestLog!A:A,C[row])` (Total Rewards)
- Column V: `=SUMIFS(HarvestLog!F:F,HarvestLog!B:B,B[row],HarvestLog!A:A,C[row])` (Total Compounded)

These automatically sum all harvest entries where:
- HarvestLog Protocol (B) = Position Protocol (B)
- HarvestLog Pair (A) = Position Pair (C)

## Example Usage

```
User: "Add harvest for SUI/USDC Turbos: $47.75 fees, $20.68 rewards, all compounded"

Assistant workflow:
1. Reads Positions sheet, finds "Turbos2 SUI/USDC" position
2. Confirms: "Recording harvest for Turbos2 SUI/USDC"
3. Calculates: Total = $68.43, Compounded = $68.43, Remaining = $0
4. Shows preview of HarvestLog entry
5. Appends: ["SUI/USDC", "Turbos2", "12/27/2025", "$47.75", "$20.68", "$68.43", "$0"]
6. Confirms success and notes that Position formulas will auto-update
```

```
User pastes raw Orca UI:
SOL / USDC
0.04%
In Range
Balance $12,078.25
Pending Yield $70.84
Harvest Yield

Assistant parses:
- Pair: SOL/USDC
- Fees to harvest: $70.84
- Rewards: $0 (none visible)

Then reads Positions sheet, finds active "Orca3 SOL/USDC".
Asks: "Parsed $70.84 pending yield from Orca SOL/USDC (position: Orca3). Was this all withdrawn, all compounded, or split?"
User answers → shows preview → confirms → writes HarvestLog row.
```
