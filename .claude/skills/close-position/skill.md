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

### 2. Confirm Data
Display a preview of what will be updated:
- Closed status: N → Y
- Final Amount TokenA (Column X)
- Final Amount TokenB (Column Y)
- Current Price TokenA (Column L)
- Current Price TokenB (Column M)

### 3. Update the Position
Use `mcp__google-sheets__update_cells` to update columns L, M, W, X, Y:
- spreadsheet_id: `1DgpluaBYRlprHmmxl8XkaU58SxGgOkXqSUkqfzb653c`
- sheet: "Positions"
- range: `L{N}:M{N}` (where N is the position row number)
- data: [[Current Price TokenA, Current Price TokenB]]

Then update the status and final amounts:
- range: `W{N}:Y{N}`
- data: [["Y", Final Amount TokenA, Final Amount TokenB]]

### 4. Report Results
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