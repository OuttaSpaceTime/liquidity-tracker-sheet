# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository manages a **Liquidity Pools Tracker** Google Sheet through Claude Code skills and the Google Sheets MCP server. The tracker monitors cryptocurrency liquidity pool positions across multiple protocols (Orca, Aero, Turbos) and networks (Solana, Base, Sui), tracking entry/exit values, fees, rewards, and P&L calculations.

**Sheet ID**: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`

## Sheet Structure

### Positions Sheet (Main Tab)
Columns A-Z track individual liquidity pool positions:

**Core Position Data (A-K):**
- A: Date Entered
- B: Protocol (Orca, Aero, Turbos, etc.)
- C: Pair (SOL/USDC, ETH/USDC, SUI/USDC)
- D: Network (Solana, Base, Sui)
- E: Range (low-high price range)
- F-I: Entry prices and amounts for TokenA/TokenB
- J: Total Entry Value ($)
- K: Fees Paid ($)

**Current Position Data (L-R):**
- L-O: Current prices and amounts for TokenA/TokenB
- P: Accrued Fees (unharvested $)
- Q-S: Reward token details and unharvested amount

**Harvested & Status (T-V):**
- T: Total Harvested Fees ($) - SUMIFS formula from HarvestLog
- U: Total Harvested Rewards (EUR)
- V: Total Compounded - tracks reinvested amounts
- W: Closed (Y/N) - position status

**Exit Data (X-Y):**
- X: Final Amount TokenA (for closed positions)
- Y: Final Amount TokenB (for closed positions)

**Calculated Values (Z):**
- Z: LP Underlying Value ($) - current value of LP position

### HarvestLog Sheet
Tracks all fee/reward harvesting transactions:

**Columns:**
- A: LP Pair
- B: Protocol
- C: Date
- D: Fees Harvested ($)
- E: Rewards Harvested ($)
- F: Compounded - amount reinvested into position
- G: Remaining - amount withdrawn

The Positions sheet references this data via SUMIFS formulas in columns T and U.

## Available Skills

All skills are located in `.claude/skills/` and work with the Google Sheets MCP server:

### Core Skills
- **list-sheets**: List all Google Sheets in your Drive
- **find-sheet**: Find a specific sheet by name (partial matching supported)
- **sheet-info**: Get detailed metadata about a spreadsheet
- **read-sheet**: Read data from specific ranges (A1 notation)
- **write-sheet**: Write/update data in specific ranges
- **append-sheet**: Append new rows without overwriting
- **clear-range**: Clear data from specific ranges (requires confirmation)
- **search-sheets**: Advanced filtering by name, owner, date, sharing status

### Specialized Skills
- **read-liquidity**: Analyzes the Liquidity Pools Tracker and provides:
  - Overview of total/active/closed positions
  - Detailed breakdown of active positions with P&L
  - Summary statistics (capital, fees, rewards, overall P&L)
  - Recent activity highlights
  - Alerts for significant gains/losses (>10%)

- **add-position**: Adds a new liquidity pool position to the Positions sheet:
  - Collects all required entry data (date, protocol, pair, network, range, prices, amounts)
  - Automatically determines the next protocol number (e.g., Turbos → Turbos2)
  - Calculates total entry value automatically
  - Creates proper SUMIFS formulas for columns T, U, V (harvested totals)
  - Creates LP Underlying Value formula for column Z
  - Adjusts all formula row numbers to match the new position row

- **harvest**: Records harvested fees and rewards to the HarvestLog sheet:
  - Identifies the correct protocol identifier by reading active positions
  - Collects fees harvested, rewards harvested amounts
  - Tracks whether amounts were compounded (reinvested) or withdrawn
  - Ensures protocol identifier matches exactly with Positions sheet for SUMIFS formulas
  - Automatically updates Position sheet totals via existing formulas

- **close-position**: Closes a liquidity pool position by updating all required fields:
  - Finds the position by protocol and pair
  - Updates Closed status (column W) to "Y"
  - Records final token amounts (columns X-Y)
  - Updates current prices to final prices (columns L-M)
  - Ensures LP Underlying Value formula uses final amounts

## Common Operations

### Adding New Position Entry
1. Use `add-position` skill to add a new liquidity pool position
2. Provide: Protocol, Pair, Network, Range, Entry prices, Entry amounts
3. The skill will:
   - Calculate total entry value automatically
   - Add all required formulas (columns T, U, V, Z) with correct row numbers
   - Set initial status as active (Closed = "N")
   - Default current values to entry values

### Recording Harvested Fees/Rewards
1. Use `harvest` skill to record fee and reward harvests
2. Provide: Pair, Protocol (or let skill identify it), Fees, Rewards amounts
3. Specify if compounded (reinvested), withdrawn, or split between both
4. The skill will:
   - Verify the exact protocol identifier from Positions sheet
   - Calculate Compounded and Remaining amounts
   - Add entry to HarvestLog with matching protocol identifier
   - Position sheet totals auto-update via SUMIFS formulas

### Closing a Position
1. Use `close-position` skill with protocol, pair, final amounts, and final prices
2. The skill will update all required fields:
   - Column W (Closed): "Y"
   - Columns X-Y: Final token amounts at withdrawal
   - Columns L-M: Current prices at time of closing
3. LP Underlying Value formula automatically switches to use final amounts

### Analyzing Portfolio
1. Use `read-liquidity` skill for comprehensive overview
2. Or use `read-sheet` with range "Positions!A5:Z100" for raw data

## Implementation Reference

### Planned Enhancement (IMPLEMENTATION_PLAN.md)
The implementation plan outlines adding proper tracking of compounded vs withdrawn fees/rewards to accurately calculate ROI. Key changes include:

- Adding "Total Fees Compounded" and "Total Rewards Compounded" columns
- New "Adjusted Entry Value" that includes reinvested amounts
- Updated Net PnL and ROI formulas to properly account for compounding
- Distinction between realized profit (withdrawn) and unrealized profit (in pool)

This plan is detailed but NOT yet implemented. The current sheet structure (as described above) is what's live.

## MCP Server Configuration

Located in `.mcp.json`:
```json
{
  "mcpServers": {
    "google-sheets": {
      "type": "stdio",
      "command": "/home/felix/Code/Misc/claude-google-sheets-mcp/venv/bin/python",
      "args": ["-m", "claude_google_sheets.server"]
    }
  }
}
```

The Google Sheets MCP server provides these tools (with full names):
- `mcp__google-sheets__list_spreadsheets` - List all spreadsheets
- `mcp__google-sheets__list_sheets` - List all sheets in a spreadsheet
- `mcp__google-sheets__get_sheet_data` - Read data from a sheet
- `mcp__google-sheets__get_sheet_formulas` - Read formulas from a sheet
- `mcp__google-sheets__update_cells` - Update/write data to cells
- `mcp__google-sheets__batch_update_cells` - Batch update multiple ranges
- `mcp__google-sheets__batch_update` - Advanced batch operations
- `mcp__google-sheets__add_rows` - Add rows to a sheet
- `mcp__google-sheets__add_columns` - Add columns to a sheet
- `mcp__google-sheets__create_spreadsheet` - Create a new spreadsheet
- `mcp__google-sheets__create_sheet` - Create a new sheet tab
- `mcp__google-sheets__copy_sheet` - Copy sheet between spreadsheets
- `mcp__google-sheets__rename_sheet` - Rename a sheet
- `mcp__google-sheets__share_spreadsheet` - Share a spreadsheet
- `mcp__google-sheets__list_folders` - List Google Drive folders
- `mcp__google-sheets__get_multiple_sheet_data` - Read from multiple ranges
- `mcp__google-sheets__get_multiple_spreadsheet_summary` - Get summaries of multiple spreadsheets

All skills wrap these MCP tools with user-friendly workflows.

## Slash Commands

Quick access commands located in `.claude/commands/`:

- **/add-position** - Add a new liquidity pool position (invokes add-position skill)
- **/harvest** - Record harvested fees and rewards (invokes harvest skill)
- **/close-position** - Close a liquidity pool position (invokes close-position skill)
- **/portfolio** - Analyze portfolio and positions (invokes read-liquidity skill)

These commands provide quick shortcuts to the most common operations.

## Key Formulas

### Total Harvested Fees (Column T)
```
=SUMIFS(HarvestLog!$D:$D,HarvestLog!$B:$B,B6,HarvestLog!$A:$A,C6)
```
Sums fees from HarvestLog matching Protocol (B) and Pair (C)

### LP Underlying Value (Column Z)
```
=IF(W6="Y",(X6*L6)+(Y6*M6),(N6*L6)+(O6*M6))
```
Uses final amounts if closed (W="Y"), otherwise current amounts

## Working with This Repository

### Reading Data
- Prefer `read-liquidity` skill for analysis
- Use `mcp__google-sheets__get_sheet_data` for specific ranges
- Set include_grid_data: false to get formatted values only (more efficient)
- Set include_grid_data: true only when you need formatting metadata

### Writing Data
- Use `mcp__google-sheets__update_cells` to write data to specific ranges
- Determine next row number first by reading the sheet
- Formulas should be entered as strings and will be interpreted by Google Sheets
- Always preview data before confirming writes

### Formula Updates
- Use `mcp__google-sheets__get_sheet_formulas` to read formulas
- Update formulas in row 6 first, then copy down
- Work RIGHT to LEFT when shifting columns to avoid reference confusion

### Data Validation
- Closed positions must have "Y" in column W and values in X-Y
- Active positions have "N" in column W
- All currency values should include $ or EUR symbol
- Dates should be in M/D/YYYY format
