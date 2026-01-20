# Find Sheet

Find a Google Sheet by name with partial matching support.

## Instructions

When the user wants to find a specific sheet by name, follow this workflow.

## Workflow

### 1. Get Search Term
Ask the user for the sheet name to search for (if not provided).

### 2. List All Spreadsheets
Use `mcp__google-sheets__list_spreadsheets` to get all available spreadsheets.

### 3. Filter Results
Filter the results by matching the sheet name:
- Use case-insensitive partial matching
- Match against the title field
- Include all sheets that contain the search term

### 4. Display Results
Show the matching results with:
- Sheet name/title
- Sheet ID
- URL: `https://docs.google.com/spreadsheets/d/{ID}`

Format:
```
Found N matching sheet(s):

1. **[Sheet Name]**
   - ID: [spreadsheet_id]
   - URL: https://docs.google.com/spreadsheets/d/[spreadsheet_id]

2. **[Sheet Name]**
   - ID: [spreadsheet_id]
   - URL: https://docs.google.com/spreadsheets/d/[spreadsheet_id]
```

### 5. Handle Multiple Matches
If multiple matches are found:
- Show all matching sheets
- Ask the user to specify which one they want to work with
- User can select by number or provide more specific search terms

## Example Usage

```
User: "Find the liquidity tracker sheet"