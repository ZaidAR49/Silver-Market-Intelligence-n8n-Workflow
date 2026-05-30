# 🥈 Silver Market Intelligence — n8n Workflow README

> Automated daily silver market briefing delivered to Telegram, in Arabic.

---

## 1. Overview

This n8n workflow delivers an automated, AI-powered silver market intelligence brief to a Telegram chat every weekday morning. It pulls live price data and news headlines, runs them through an OpenAI language model, and produces a structured Arabic-language analysis — with an optional high-volatility alert when prices make a significant move.

| Property | Value |
|---|---|
| **Trigger** | Weekdays (Mon–Fri) at 08:00 Amman time (Asia/Amman, UTC+3) |
| **Output** | Telegram message — full AI report, plus price alert if \|Δ\| ≥ 2.5% |
| **Language** | Arabic (configurable in system prompt) |
| **Currency** | USD and JOD (Jordanian Dinar, default rate: 0.709) |

---

## 2. Data Sources

Three sources are fetched in parallel on every run:

### Silver Futures Price — Yahoo Finance API
```
GET https://query1.finance.yahoo.com/v8/finance/chart/SI=F?interval=1d&range=5d
```
Fields used: `regularMarketPrice`, `chartPreviousClose`, `fiftyTwoWeekHigh`, `fiftyTwoWeekLow`

### Yahoo Finance Silver RSS
```
GET https://finance.yahoo.com/rss/headline?s=SI=F
```
Returns up to 7 recent silver-specific headlines with title, description, and publish date.

### Investing.com Commodities RSS
```
GET https://www.investing.com/rss/news_25.rss
```
Returns up to 7 recent commodities news items with title, description, and publish date.

---

## 3. Workflow Step-by-Step

| # | Node | Type | Description |
|---|---|---|---|
| 1 | **Daily 8AM Mon–Fri** | Schedule Trigger | Fires Mon–Fri at 08:00 Amman time. Kicks off all three parallel data fetches simultaneously. |
| 2 | **Get Silver Price1** | HTTP Request | Calls the Yahoo Finance chart API to retrieve current spot price, previous close, and 52-week high/low for silver futures. |
| 3 | **Yahoo Finance Silver RSS** | HTTP Request | Fetches recent silver headlines from Yahoo Finance's RSS endpoint. Returns raw XML parsed in the Code node. |
| 4 | **Investing.com Commodities RSS** | HTTP Request | Fetches commodities news from Investing.com's RSS feed. Returns raw XML parsed in the Code node. |
| 5 | **Merge** | Merge | Waits for all three parallel branches to complete, then merges into a single payload for processing. |
| 6 | **Build AI Prompt1** | Code (JS) | Parses XML from both RSS feeds, formats price data, converts to JOD, and assembles the full structured prompt for the AI. |
| 7 | **Basic LLM Chain** | LangChain LLM Chain | Sends the assembled prompt to the configured OpenAI model with the Arabic analysis system prompt. |
| 8 | **If** | Condition | Checks whether `Math.abs(priceChangePct) >= 2.5`. True → sends alert + daily report. False → sends daily report only. |
| 9a | **alert** | Telegram | Sends a concise rule-based price alert with price, JOD equivalent, change %, and 52W range. No AI involved. |
| 9b | **Send Daily Report1** | Telegram | Sends the full AI-generated Arabic analysis report to the configured chat ID. |

---

## 4. AI Analysis Structure

The LLM produces a full Arabic report using this exact format:

```
🥈 *تقرير الفضة اليومي*
📅 <التاريخ>
━━━━━━━━━━━━━━━━━━━━━━

📊 *ملخص السعر*
💰 السعر الحالي: $<price> / أوقية
🇯🇴 بالدينار: <price × 0.709> دينار / أوقية
📉 التغيير اليومي: <change>%
📈 أعلى سعر 52 أسبوع: $<high52w>
📉 أدنى سعر 52 أسبوع: $<low52w>
━━━━━━━━━━━━━━━━━━━━━━

🎯 *توجه السوق:* صاعد 🟢 / محايد 🟡 / هابط 🔴
━━━━━━━━━━━━━━━━━━━━━━
⚖️ *توصية التداول:* [شراء 🟢 / بيع 🔴 / احتفاظ 🟡]
🎯 *السعر المستهدف للتنفيذ:* $<suggested price>
━━━━━━━━━━━━━━━━━━━━━━

📰 *أبرز الأخبار المؤثرة*
1️⃣ <headline 1 + impact>
2️⃣ <headline 2 + impact>
3️⃣ <headline 3 + impact>
━━━━━━━━━━━━━━━━━━━━━━

🔮 *التوقعات قصيرة المدى*
<1–2 sentence forecast>
━━━━━━━━━━━━━━━━━━━━━━
⚠️ _هذا التقرير للأغراض المعلوماتية فقط، وليس نصيحة استثمارية._
```

### Alert Severity Logic

The model classifies each report as:

- **`urgent`** — price change ≥ 2.5% OR price within 2% of 52W high or low
- **`info`** — all other normal market conditions

---

## 5. Telegram Output Formats

| Message Type | Trigger | Content |
|---|---|---|
| **Daily Report** | Every weekday 8AM | Full AI Arabic analysis — price summary, market direction, trade recommendation, top news, short-term forecast |
| **Price Alert** | When \|Δ\| ≥ 2.5% | Concise rule-based alert with price in USD & JOD, % change, 52W range. Sent *before* the daily report on alert days. |

### Price Alert template

```
🚨 *SILVER PRICE ALERT — BIG MOVE DETECTED*
📅 <date>
━━━━━━━━━━━━━━━━━━━━━━
💰 *Price:* $<price>/oz
🇯🇴 *JOD:* <price × 0.709> JOD/oz
📊 *Change:* <change>%
━━━━━━━━━━━━━━━━━━━━━━
📈 *52W High:* $<high>/oz
📉 *52W Low:* $<low>/oz
━━━━━━━━━━━━━━━━━━━━━━
⚠️ Rule-based alert — no AI involved. Verify manually.
```

---

## 6. Setup Instructions

### Step 1 — Import the Workflow

1. Open your n8n instance and go to **Workflows → Import from File**.
2. Select the workflow JSON file and confirm the import.
3. The workflow will appear with all nodes pre-configured.

### Step 2 — Add Credentials

**OpenAI API Key**
1. In n8n, go to **Settings → Credentials → Add Credential**.
2. Choose **OpenAI API** and paste your key.
3. Open the `OpenAI GPT-4o1` node and select the credential.

**Telegram Bot Token**
1. Create a bot via [@BotFather](https://t.me/BotFather) on Telegram. Copy the bot token.
2. In n8n, add a **Telegram API** credential with the token.
3. Open both Telegram nodes (`Send Daily Report1` and `alert`) and assign the credential.

### Step 3 — Set Your Chat ID

Both Telegram nodes are pre-set to chat ID `2076665816`. Replace this with your own:

- **Personal chat ID** — message [@userinfobot](https://t.me/userinfobot) to get your ID.
- **Group chat ID** — add @userinfobot to the group; it will report the group ID.
- Update the `chatId` field in **both** the `Send Daily Report1` node and the `alert` node.

### Step 4 — Activate

1. Click the toggle in the top-right of the workflow editor to set it **Active**.
2. The schedule trigger will fire automatically at 8AM Amman time on weekdays.
3. To test immediately, click **Execute Workflow**.

---

## 7. Configuration Reference

| Setting | Where | Default | Notes |
|---|---|---|---|
| Telegram Chat ID | Both Telegram nodes | `2076665816` | Your own chat or group ID |
| Alert threshold | `If` node | `2.5` | Min % change to trigger price alert |
| JOD exchange rate | `Build AI Prompt1` + Telegram messages | `0.709` | Update to current JOD/USD rate |
| Cron schedule | `Daily 8AM Mon–Fri` trigger | `0 8 * * 1-5` | Any valid cron expression |
| AI model | `OpenAI GPT-4o1` node | `gpt-5-chat-latest` | Any OpenAI chat model |
| Report language | System prompt | Arabic | Rewrite the format template in Section 0 |

---

## 8. System Prompt Notes

The system prompt in the `Basic LLM Chain` node contains two operational modes:

### Section 0 — Market Data Processing (active mode)

Handles the case where raw market data is sent as the user message. The model acts as the **master Silver Market Analyst** and produces the full Arabic report from price data and news headlines. This is the mode used by the current workflow.

### Sections 1–8 — Alert Agent Mode (extensibility)

Handles a structured JSON alert payload. The model acts as a routing agent, dispatches alerts to Telegram or Gmail, and returns a strict JSON confirmation object. This is designed for future extensibility (e.g., adding email delivery).

### Customising the Output

- **Change language** — rewrite the Arabic formatting template in Section 0 of the system prompt.
- **Add report sections** — extend the numbered structure in the Section 0 format block.
- **Update JOD rate** — find `0.709` in the `Build AI Prompt1` Code node and both Telegram message templates.
- **Adjust max length** — increase `maxTokens` beyond `2500` in the `OpenAI GPT-4o1` node if reports are getting cut off.

---

## 9. Troubleshooting

### No message received on Telegram
- Check that the Telegram credential is correctly assigned in both Telegram nodes.
- Verify the Chat ID is correct — use @userinfobot to confirm.
- Ensure you have sent `/start` to the bot before expecting messages.

### Price shows as `N/A` or `Unavailable`
- Yahoo Finance's API occasionally rate-limits or changes its response format.
- Run the workflow manually and inspect the `Get Silver Price1` node output.
- Check that `chart.result[0].meta` exists in the response; if empty, wait a few minutes and retry.

### RSS feeds return empty news
- The RSS parser handles both CDATA-wrapped and plain-text titles/descriptions.
- If a feed changes its XML structure, update the regex patterns in the `Build AI Prompt1` Code node.
- Check whether the RSS URLs are accessible from your n8n host — some feeds block certain server IPs.

### AI response is malformed or truncated
- Lower the `temperature` (currently `0.2`) for more deterministic output.
- If the model returns JSON instead of the report, check the system prompt for conflicting Section 0/1 instructions.
- Increase `maxTokens` beyond `2500` if the report is being cut off.

---

## 10. Disclaimer

> ⚠️ **This workflow is for informational and educational purposes only.**
> Nothing produced by this automation constitutes financial or investment advice.
> Silver trading carries significant risk. Always conduct your own research and
> consult a qualified financial advisor before making any trading decisions.

---

*Silver Market Intelligence • n8n Workflow Documentation*
