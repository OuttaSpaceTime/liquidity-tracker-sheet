---
name: read-liquidity
description: Read and analyze the Liquidity Pools Tracker sheet, providing an overview of open and closed positions with key metrics.
---

# Read Liquidity Pools Tracker

Quickly read and analyze your Liquidity Pools Tracker sheet.

## Instructions

Follow these steps to provide a comprehensive overview:

1. **Find the sheet**: Use the spreadsheet ID `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c` (Liquidity Pools Tracker 2026)

2. **Read the data**: Use `mcp__google-sheets__get_sheet_data` with:
   - spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
   - sheet: "Positions"
   - range: "A1:Z1000"
   - include_grid_data: false (returns formatted values only)

3. **Analyze and present the data** in this format:

## Overview
- Total positions tracked: [count]
- Active positions (Closed = N): [count]
- Closed positions (Closed = Y): [count]

## Active Positions
For each active position (where Closed = N), show:
- **[Protocol]** - [Pair] on [Network] ([Date Entered])
  - Range: [Range]
  - Entry Value: [Total Entry Value]
  - Current Value: [LP Underlying Value]
  - P&L: [calculate difference and percentage]
  - Fees Harvested: [Total Harvested Fees]
  - Rewards Harvested: [Total Harvested Rewards]

## Summary Statistics
- Total Active Capital: [sum of LP Underlying Value for active positions]
- Total Fees Harvested (all positions): [sum]
- Total Rewards Harvested (all positions): [sum]
- Overall P&L: [calculate total current value vs total entry value for active positions]

## Recent Activity
Show the 3 most recently entered positions (by Date Entered)

## Tips
- Highlight any positions with significant unrealized P&L (>10% gain or loss)
- Note if any active positions have unharvested fees/rewards available
