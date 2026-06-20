# AvtoPro Parser Bot

A professional Telegram bot for automated parsing of auto parts from [avto.pro](https://avto.pro), with results saved directly to Google Sheets. Supports parallel parsing with multiple workers, cookie-based authentication, and full session history tracking.

## Features

### Core Functionality

- **Telegram Bot Interface** — Start parsing with a single button click
- **Selenium-Based Scraper** — Handles dynamic pages, modals, and pagination automatically
- **Multi-Worker Parallel Parsing** — Process multiple part numbers simultaneously (default: 3 workers)
- **Google Sheets Integration** — Saves results to a live spreadsheet with history tracking
- **Smart Page Detection** — Automatically handles three page types: single card, list with pagination, and info pages
- **UAH Currency Filter** — Skips offers with negotiable or non-UAH prices
- **Cookie Authentication** — Uses saved browser cookies to maintain a logged-in session
- **Detailed Logging** — Per-worker logs with timestamps saved to the `logs/` directory

---

## Requirements

- Python 3.8+
- Google Chrome + ChromeDriver
- Telegram Bot Token
- Google Service Account with Sheets & Drive API access

---

## Installation

**1. Clone the repository:**

```bash
git clone https://github.com/fedyaqq34356/AvtoParser.git
cd AvtoParser
```

**2. Create and activate a virtual environment:**

```bash
python -m venv venv
```

Windows:

```bash
venv\Scripts\activate
```

macOS / Linux:

```bash
source venv/bin/activate
```

**3. Install dependencies:**

```bash
pip install -r requirements.txt
```

---

## Configuration

### 1. Environment Variables

Create a `.env` file in the project root:

```env
BOT_TOKEN=your_telegram_bot_token_here
```

Getting Bot Token:

- Message [@BotFather](https://t.me/BotFather) on Telegram
- Send `/newbot` command
- Follow instructions to create your bot
- Copy the token to `.env` file

---

### 2. Google Sheets (`config.json`)

Create `config.json` in the project root:

```json
{
  "spreadsheet_id": "your_google_spreadsheet_id_here"
}
```

The spreadsheet ID is found in the Google Sheets URL:
`https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>/edit`

---

### 3. Google Service Account (`credentials.json`)

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project (or use an existing one)
3. Enable the **Google Sheets API** and **Google Drive API**
4. Create a **Service Account** and download the JSON key file
5. Rename it to `credentials.json` and place it in the project root
6. Share your Google Spreadsheet with the service account email (found inside `credentials.json` as `client_email`), granting **Editor** access

---

### 4. Browser Cookies (`cookie.json`)

The parser uses saved cookies to access avto.pro as a logged-in user.

1. Log in to [avto.pro](https://avto.pro) in Chrome
2. Use a browser extension such as [EditThisCookie](https://chrome.google.com/webstore/detail/editthiscookie/) to export cookies
3. Save the exported JSON as `cookie.json` in the project root

---

### 5. ChromeDriver

By default the parser looks for ChromeDriver at `/usr/local/bin/chromedriver`. You can override this with an environment variable:

```env
CHROMEDRIVER_PATH=/path/to/your/chromedriver
```

Make sure your ChromeDriver version matches your installed Chrome version. Check [chromedriver.chromium.org](https://chromedriver.chromium.org/downloads) for downloads.

---

## Google Spreadsheet Structure

The bot reads part numbers from a sheet named **`Номери для парсингу`**, column A (one number per row).

It writes results to two sheets automatically:

| Sheet | Description |
|---|---|
| `Готова таблиця` | Latest parsing results (overwritten each run) |
| `Історія` | Cumulative history of all past runs |

Both sheets use these columns:

`номер` · `виробник` · `код` · `опис` · `доставка` · `місто` · `ціна` · `наявність` · `магазин` · `дата парсингу`

---

## Usage

### Starting the Bot

Run the main script:

```bash
python main.py
```

Console output:

```
2026-02-19 14:30:12 [INFO] bot: Бот запущен
```

### Bot Commands

**For Admins:**

- `/start` — Open main menu and start parsing

**Parsing Flow:**

1. Bot connects to your Google Spreadsheet
2. Loads all part numbers from the `Номери для парсингу` sheet
3. Spins up parallel workers to parse each number on avto.pro
4. Saves all results (and failed lookups) to both output sheets
5. Replies with a summary: total offers, successful and failed numbers

---

## How It Works

```
Telegram /start
    ↓
Load part numbers from Google Sheets
    ↓
Spawn N parallel workers (ThreadPoolExecutor)
    ↓
Each worker:
  → Load cookies → Open avto.pro → Search part number
  → Detect page type (card / list / info page)
  → Parse all offers (with pagination support)
  → Filter non-UAH and negotiable prices
  → Open seller modal to get shop name & confirmed price
    ↓
Aggregate all results
    ↓
Save to "Готова таблиця" + append to "Історія"
    ↓
Send summary message to Telegram
```

---

## Parsed Data Fields

| Field | Description |
|---|---|
| `number` | Part number queried |
| `maker` | Manufacturer / brand |
| `code` | Part code as listed on site |
| `description` | Product description (max 120 chars) |
| `delivery` | Delivery timeframe |
| `city` | Seller city |
| `price` | Price in UAH |
| `availability` | In stock status |
| `shop_name` | Seller shop name |
| `parse_date` | Date of parsing |

---

## Project Structure

```
AvtoParser/
├── main.py              # Entry point
├── bot.py               # Telegram bot logic and parsing orchestration
├── parser.py            # Selenium scraper (AvtoProParser class)
├── worker.py            # Worker function for parallel processing
├── google_sheets.py     # Google Sheets read/write manager
├── logger.py            # Logging configuration
├── utils.py             # Cookie loading utility
├── requirements.txt     # Python dependencies
├── .env                 # Environment variables (create manually)
├── config.json          # Spreadsheet ID config (create manually)
├── cookie.json          # Exported browser cookies (create manually)
└── credentials.json     # Google service account credentials (create manually)
```

---

## Logging

Each run creates a new timestamped log file in the `logs/` directory:

```
logs/parser_20260219_143012.log
```

Logs include per-worker step-by-step output: page type detection, row parsing, price validation, modal interaction, and error traces.

---

## Troubleshooting

**Bot says parsing is already running**
Only one parsing session runs at a time. Wait for the current session to finish before starting another.

**`SessionNotCreatedException` / ChromeDriver error**
Your ChromeDriver version doesn't match Chrome. Download the matching version from [chromedriver.chromium.org](https://chromedriver.chromium.org/downloads) or set `CHROMEDRIVER_PATH` to the correct binary.

**Google Sheets `APIError` / permission denied**
Make sure you shared the spreadsheet with the service account email from `credentials.json` and granted **Editor** access.

**Cookies not working / redirected to login**
Your `cookie.json` has expired. Log back in to avto.pro, re-export the cookies, and replace the file.

**No results for a part number**
The bot logs a warning and writes a blank row for that number in the output sheet. Check the log file for details.

---

## Dependencies

| Package | Purpose |
|---|---|
| `aiogram` | Telegram Bot API framework |
| `selenium` | Browser automation |
| `gspread` | Google Sheets API client |
| `oauth2client` | Google service account auth |
| `python-dotenv` | `.env` file support |
| `webdriver-manager` | ChromeDriver management helper |

---

## Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/improvement`)
3. Commit changes (`git commit -am 'Add new feature'`)
4. Push to branch (`git push origin feature/improvement`)
5. Create Pull Request

---

## Support

For issues, questions, or suggestions:

- **GitHub Issues:** https://github.com/fedyaqq34356/AvtoParser/issues
- **Repository:** https://github.com/fedyaqq34356/AvtoParser.git

---

## License

This project is licensed under the **GNU General Public License v3.0** (GPL-3.0).

You are free to use, modify, and distribute this software under the terms of the GPL-3.0. Any derivative works must also be distributed under the same license. Full license text: [gnu.org/licenses/gpl-3.0](https://www.gnu.org/licenses/gpl-3.0.html)

---

Made with ❤️ for community