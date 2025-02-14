ensoka_bot.py
import logging
import aiohttp  # Async HTTP for faster API calls
import openai
import os
import time
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, CallbackContext

# 🔹 Load API Keys from Environment Variables
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")

# Pump.fun API URL
PUMPFUN_API_URL = "https://pumpapi.fun/api"

# OpenAI Setup
openai.api_key = OPENAI_API_KEY

# Logging
logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)

# 🔹 API Response Caching (Reduces API Costs)
cache = {}

def get_cached_data(contract_address, cache_duration=60):
    """Check if token data is cached to reduce API requests."""
    if contract_address in cache:
        data, timestamp = cache[contract_address]
        if time.time() - timestamp < cache_duration:
            return data  # Return cached data if within time limit
    return None

def cache_data(contract_address, data):
    """Store token data in cache with timestamp."""
    cache[contract_address] = (data, time.time())

# 🔹 Async API Calls (Faster Response Times)
async def fetch_data(url):
    """Fetch data asynchronously from an API."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def fetch_pumpfun_data(contract_address):
    """Fetch token data asynchronously from Pump.fun API."""
    cached_response = get_cached_data(contract_address)
    if cached_response:
        return cached_response

    url = f"{PUMPFUN_API_URL}/token/{contract_address}"
    data = await fetch_data(url)
    cache_data(contract_address, data)
    return data

async def fetch_whale_trades(contract_address):
    """Fetch whale trades asynchronously."""
    url = f"{PUMPFUN_API_URL}/trades/{contract_address}"
    return await fetch_data(url)

# 🔹 Rug Pull Detection
def detect_rug_risk(liquidity, volume_24h, holders):
    rug_score = 0
    reasons = []

    if liquidity < 5000:
        rug_score += 3
        reasons.append("🔴 **Very Low Liquidity (<$5,000)** - High rug risk.")

    if volume_24h > liquidity * 10:
        rug_score += 3
        reasons.append("🔶 **High Trading Volume vs. Low Liquidity** - Possible pump & dump.")

    if holders < 50:
        rug_score += 2
        reasons.append("🟠 **Low Number of Holders (<50)** - Risk of centralization.")

    if rug_score >= 5:
        return f"🚨 **High Rug Risk!** 🚨\n" + "\n".join(reasons)
    elif rug_score >= 3:
        return f"⚠️ **Medium Rug Risk.** Caution advised.\n" + "\n".join(reasons)
    else:
        return "✅ **Low Rug Risk** - No major red flags detected."

# 🔹 Whale Activity Tracking
def detect_whale_activity(transactions):
    whale_threshold = 5000
    whale_alerts = []

    for tx in transactions:
        amount = tx.get("amount", 0)
        wallet = tx.get("wallet", "Unknown Wallet")
        tx_type = "🟢 BUY" if tx.get("type") == "buy" else "🔴 SELL"

        if amount > whale_threshold:
            whale_alerts.append(f"🐋 **Whale {tx_type} Alert:** ${amount:,} by `{wallet}`")

    return "\n".join(whale_alerts) if whale_alerts else "✅ **No whale activity detected.**"

# 🔹 Telegram Bot Commands
async def start(update: Update, context: CallbackContext) -> None:
    """Start command handler."""
    welcome_message = (
        "🚀 **Welcome to Ensoka - Pump.fun Whale & Rug Tracker!**\n\n"
        "🔹 Send a **Pump.fun contract address**, and I'll analyze:\n"
        "✅ **Whale Activity** (Large Buy/Sell Alerts)\n"
        "✅ **Rug Pull Risk Analysis**\n"
        "✅ **Liquidity & Trading Volume**\n"
        "✅ **AI-Powered Market Insights**"
    )
    await update.message.reply_text(welcome_message, parse_mode="Markdown")

async def analyze_pumpfun_contract(update: Update, context: CallbackContext) -> None:
    """Fetches and analyzes Pump.fun token data."""
    contract_address = update.message.text.strip()

    if len(contract_address) < 30:
        await update.message.reply_text("⚠️ **Invalid contract address!** Please provide a valid Pump.fun contract.")
        return

    try:
        # Fetch data
        token_data = await fetch_pumpfun_data(contract_address)
        whale_transactions = await fetch_whale_trades(contract_address)

        if "error" in token_data:
            await update.message.reply_text("⚠️ **Token not found or invalid contract.**")
            return
        
        # Extract token details
        data = token_data.get("data", {})
        token_name = data.get("name", "Unknown")
        token_symbol = data.get("symbol", "N/A")
        price = data.get("price", 0)
        volume_24h = data.get("volume_24h", 0)
        liquidity = data.get("liquidity", 0)
        market_cap = data.get("market_cap", 0)
        holders = data.get("holders", 0)

        # Analyze Risk & Whale Activity
        rug_analysis = detect_rug_risk(liquidity, volume_24h, holders)
        whale_alerts = detect_whale_activity(whale_transactions.get("transactions", []))

        # AI Trading Insights
        ai_analysis = "Not Available"  # Placeholder to save costs on OpenAI API

        # Response Message
        response_message = (
            f"🟢 **Token Analysis - {token_name} ({token_symbol})** 🟢\n\n"
            f"💰 **Price:** `${price}`\n"
            f"📊 **24h Volume:** `${volume_24h:,}`\n"
            f"🔄 **Liquidity:** `${liquidity:,}`\n"
            f"🌍 **Market Cap:** `${market_cap:,}`\n"
            f"👥 **Holders:** `{holders}`\n\n"
            f"🔥 **Rug Pull Risk:**\n{rug_analysis}\n\n"
            f"🐋 **Whale Activity:**\n{whale_alerts}\n\n"
            f"📈 **AI Market Insight:**\n_{ai_analysis}_"
        )
        
        await update.message.reply_text(response_message, parse_mode="Markdown")

    except Exception as e:
        await update.message.reply_text(f"⚠️ **Error fetching token data:** `{str(e)}`")

# Start Bot
def main():
    app = Application.builder().token(TELEGRAM_BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, analyze_pumpfun_contract))
    app.run_polling()

if __name__ == "__main__":
    main()
