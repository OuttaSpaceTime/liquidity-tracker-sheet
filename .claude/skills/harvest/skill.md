# Harvest

Record harvested fees and rewards to the HarvestLog sheet for a liquidity pool position.

## Instructions

When recording a harvest, I need the following information:

### Required Data
1. **LP Pair** (e.g., SOL/USDC, ETH/USDC, SUI/USDC)
2. **Protocol identifier** (e.g., Turbos2, Orca10, Aero5) - must match exactly with Positions sheet
3. **Date** (default to today if not specified)
4. **Fees Harvested ($)** - total fees claimed in USD
5. **Rewards Harvested ($)** - total rewards claimed in USD (or EUR)
6. **How the harvest was used:**
   - Compounded (reinvested back into the position)
   - Withdrawn (taken out)
   - Or a combination of both

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

### 4. Preview the Entry
Show the user what will be added to HarvestLog:

| LP Pair | Protocol | Date | Fees Harvested ($) | Rewards Harvested ($) | Compounded | Remaining |
|---------|----------|------|-------------------|----------------------|------------|-----------|
| ... | ... | ... | ... | ... | ... | ... |

### 5. Append to HarvestLog
First, determine the next row in HarvestLog using `mcp__google-sheets__get_sheet_data`:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: "HarvestLog"
- range: "A:A" (to count rows)

Then use `mcp__google-sheets__update_cells` to append the row:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: `HarvestLog`
- range: `A{N}:G{N}` (where N is the next row number)
- data: [[LP Pair, Protocol, Date, Fees ($), Rewards ($), Compounded, Remaining]]

### 6. Report Results
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
