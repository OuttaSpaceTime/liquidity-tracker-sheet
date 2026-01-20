# Liquidity Pools Tracker

A Claude Code setup to track cryptocurrency liquidity pool investments across multiple protocols and networks. Manage positions, record harvests, calculate P&L, and analyze your portfolio directly through Claude Code skills.

## Features

- **Position Tracking**: Track liquidity pool positions across multiple protocols (Orca, Aero, Turbos) and networks (Solana, Base, Sui)
- **Harvest Recording**: Log harvested fees and rewards, track compounding vs withdrawals
- **Portfolio Analysis**: Get comprehensive P&L breakdowns and performance metrics
- **Position Management**: Add new positions, close positions, update values
- **Automated Calculations**: Automatic ROI, P&L, and value tracking with formulas

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- Python 3.8+ (for Google Sheets MCP server)
- Google Cloud project with Sheets API enabled
- Google OAuth credentials

## Setup

### 1. Clone the Repository

```bash
git clone https://github.com/OuttaSpaceTime/liquidity-tracker-sheet.git
cd liquidity-tracker-sheet
```

### 2. Configure Google Sheets API

1. Create a Google Cloud project at [console.cloud.google.com](https://console.cloud.google.com)
2. Enable the Google Sheets API
3. Create OAuth 2.0 credentials (Desktop app)
4. Download credentials and save as `credentials.json` in the project root

### 3. Install Google Sheets MCP Server

Clone and set up the [Google Sheets MCP server](https://github.com/anthropics/claude-google-sheets-mcp):

```bash
# Clone the MCP server repository
git clone https://github.com/anthropics/claude-google-sheets-mcp.git
cd claude-google-sheets-mcp

# Create virtual environment and install
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -e .
```

### 4. Update MCP Configuration

Edit `.mcp.json` to point to your MCP server installation:

```json
{
  "mcpServers": {
    "google-sheets": {
      "type": "stdio",
      "command": "/path/to/claude-google-sheets-mcp/venv/bin/python",
      "args": ["-m", "claude_google_sheets.server"]
    }
  }
}
```

### 5. Create Your Tracking Sheet

1. Create a new Google Sheet or copy the template
2. Update the `CLAUDE.md` file with your Sheet ID (found in the URL)
3. Create two tabs: `Positions` and `HarvestLog`

## Usage

### Starting Claude Code

```bash
cd liquidity-tracker-sheet
claude
```

On first run, you'll be prompted to authenticate with Google and grant access to Sheets.

### Available Skills

All skills can be invoked using slash commands:

#### `/portfolio` - Analyze Portfolio
Get a comprehensive overview of your liquidity pool positions:
- Total positions (active/closed)
- Detailed P&L for each position
- Summary statistics (capital deployed, fees earned, rewards, overall P&L)
- Alerts for significant gains/losses

```
/portfolio
```

#### `/add-position` - Add New Position
Add a new liquidity pool position to the tracker:

```
/add-position
```

You'll be prompted for:
- Protocol (Orca, Aero, Turbos, etc.)
- Token pair (SOL/USDC, ETH/USDC, etc.)
- Network (Solana, Base, Sui)
- Price range
- Entry prices and amounts for both tokens

The skill automatically:
- Calculates total entry value
- Assigns protocol number (e.g., Turbos2 if Turbos exists)
- Creates formulas for harvested fees/rewards tracking
- Sets up LP value calculations

#### `/harvest` - Record Harvest
Log harvested fees and rewards:

```
/harvest
```

You'll provide:
- Protocol and pair (or let skill identify from positions)
- Fees harvested (in $)
- Rewards harvested (in $)
- Whether amounts were compounded (reinvested) or withdrawn

The skill updates the HarvestLog sheet, and position totals automatically update via formulas.

#### `/close-position` - Close Position
Close out a liquidity pool position:

```
/close-position
```

You'll provide:
- Protocol and pair to close
- Final token amounts at withdrawal
- Final token prices

The skill:
- Marks position as closed
- Records final values
- Updates LP value formula to use final amounts

### Direct Skill Invocation

You can also use skills directly in conversation:

```
Can you run the read-liquidity skill to show my portfolio?
```

```
Use the add-position skill to add my new Orca SOL/USDC position
```

## Sheet Structure

### Positions Sheet

Main tab tracking all liquidity pool positions:

| Columns | Description |
|---------|-------------|
| A-K | Entry data (date, protocol, pair, network, range, entry prices/amounts, total value, fees paid) |
| L-S | Current position (current prices/amounts, accrued fees, reward tokens) |
| T-V | Harvested totals (fees, rewards, compounded amounts) - calculated via SUMIFS from HarvestLog |
| W | Closed status (Y/N) |
| X-Y | Final amounts (for closed positions) |
| Z | LP Underlying Value (calculated based on current/final amounts) |

### HarvestLog Sheet

Tracks all fee and reward harvest transactions:

| Column | Description |
|--------|-------------|
| A | LP Pair |
| B | Protocol |
| C | Date |
| D | Fees Harvested ($) |
| E | Rewards Harvested ($) |
| F | Compounded (reinvested amount) |
| G | Remaining (withdrawn amount) |

## Common Workflows

### Adding a New Position

1. Open a new LP position on your chosen protocol
2. Note entry details (prices, amounts, fees paid, range)
3. Run `/add-position` in Claude Code
4. Provide all entry information when prompted
5. Skill creates tracking row with formulas

### Recording a Harvest

1. Harvest fees/rewards from your LP position
2. Note amounts harvested and whether you reinvested
3. Run `/harvest` in Claude Code
4. Provide harvest details
5. Position sheet automatically updates totals via formulas

### Checking Performance

1. Run `/portfolio` in Claude Code
2. Review P&L for each position
3. Check overall portfolio performance
4. Identify top/bottom performers

### Closing a Position

1. Close your LP position and withdraw
2. Note final token amounts and prices
3. Run `/close-position` in Claude Code
4. Provide closing details
5. Position marked closed with final P&L calculated

## Files & Configuration

- **CLAUDE.md**: Project documentation and instructions for Claude Code
- **.mcp.json**: MCP server configuration for Google Sheets
- **.claude/skills/**: Custom skills for liquidity pool management
  - `read-liquidity/`: Portfolio analysis
  - `add-position/`: Add new positions
  - `harvest/`: Record harvests
  - `close-position/`: Close positions
  - `find-sheet/`: Find sheets by name
- **.gitignore**: Excludes credentials and tokens from version control

## Security

Sensitive files are automatically excluded from git:
- `credentials.json` - Google OAuth credentials
- `token.json` - Google OAuth tokens
- `.env` - Environment variables
- `.claude/settings.local.json` - Local Claude settings

**Never commit these files to the repository.**

## Contributing

Feel free to open issues or submit pull requests for:
- New skill features
- Additional protocols/networks
- Sheet formula improvements
- Documentation enhancements

## License

MIT

## Support

For issues or questions:
- Open an issue on GitHub
- Check CLAUDE.md for detailed implementation notes
- Refer to skill documentation in `.claude/skills/`
