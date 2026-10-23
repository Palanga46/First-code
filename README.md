import asyncio
import json
import logging
import websockets
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# ==========================================
# CONFIGURATION & PARAMETERS
# ==========================================
DERIV_APP_ID = 1089  # Default Deriv App ID
DERIV_TOKEN = "YOUR_DERIV_API_TOKEN"  # Insert Deriv Token here

TELEGRAM_BOT_TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"  # Insert Telegram Bot Token
TELEGRAM_CHAT_ID = YOUR_CHAT_ID  # Insert Chat ID (as integer, e.g. 123456789)

# Grid Strategy Parameters (Adjustable)
SYMBOL = "R_100"               # Synthetic Index Volatility 100 (or Forex pair)
FIRST_DISTANCE_POINTS = 1000   # Points for first order distance
GRID_STEP_POINTS = 1500        # Distance between subsequent orders
TOTAL_LEVELS = 3               # Max grid orders per side
INITIAL_LOT = 0.01             # Stake or lot size
TARGET_PROFIT_USD = 5.0        # Target profit in USD
RETRY_DELAY_SEC = 5            # Delay after target profit hit before restarting

# Global State Management
is_bot_active = False
placed_buy_levels = 0
placed_sell_levels = 0
open_positions = []
pending_orders = []

logging.basicConfig(level=logging.INFO)

# ==========================================
# DERIV WEBSOCKET HANDLER
# ==========================================
async def get_current_price(ws):
    """Fetch current Ask and Bid tick prices."""
    req = {"ticks": SYMBOL}
    await ws.send(json.dumps(req))
    response = await ws.recv()
    data = json.loads(response)
    if "tick" in data:
        quote = data["tick"]["quote"]
        return quote, quote  # Simple quote reference
    return None, None

async def authorize_deriv(ws):
    """Authenticate with Deriv API."""
    auth_req = {"authorize": DERIV_TOKEN}
    await ws.send(json.dumps(auth_req))
    res = await ws.recv()
    data = json.loads(res)
    return "authorize" in data and "error" not in data

async def calculate_total_profit(ws):
    """Calculate total running profit across all open contracts/positions."""
    req = {"portfolio": 1}
    await ws.send(json.dumps(req))
    res = await ws.recv()
    data = json.loads(res)
    
    total_profit = 0.0
    if "portfolio" in data and "contracts" in data["portfolio"]:
        for contract in data["portfolio"]["contracts"]:
            total_profit += float(contract.get("profit", 0.0))
    return total_profit

async def close_all_and_cancel(ws, telegram_app):
    """Fast close for all positions and cancel pending orders."""
    global open_positions, pending_orders, placed_buy_levels, placed_sell_levels
    
    # Send request to sell all open portfolio positions
    req = {"portfolio": 1}
    await ws.send(json.dumps(req))
    res = await ws.recv()
    data = json.loads(res)
    
    if "portfolio" in data and "contracts" in data["portfolio"]:
        for contract in data["portfolio"]["contracts"]:
            contract_id = contract["contract_id"]
            sell_req = {"sell": contract_id, "price": 0}
            await ws.send(json.dumps(sell_req))
            await ws.recv()
            
    placed_buy_levels = 0
    placed_sell_levels = 0
    open_positions.clear()
    pending_orders.clear()
    
    await telegram_app.bot.send_message(
        chat_id=TELEGRAM_CHAT_ID, 
        text=f"✅ Target Profit Reached! Closed all positions. Waiting {RETRY_DELAY_SEC} seconds to restart grid..."
    )

async def grid_engine(telegram_app):
    """Main trading loop managing sequential grid and take profit target."""
    global is_bot_active, placed_buy_levels, placed_sell_levels
    
    uri = f"wss://ws.derivws.com/websockets/v3?app_id={DERIV_APP_ID}"
    
    while True:
        if not is_bot_active:
            await asyncio.sleep(2)
            continue

        try:
            async with websockets.connect(uri) as ws:
                if not await authorize_deriv(ws):
                    logging.error("Deriv Authorization Failed.")
                    await asyncio.sleep(10)
                    continue

                while is_bot_active:
                    # 1. Check Basket Target Profit
                    current_profit = await calculate_total_profit(ws)
                    if len(open_positions) > 0 and current_profit >= TARGET_PROFIT_USD:
                        await close_all_and_cancel(ws, telegram_app)
                        await asyncio.sleep(RETRY_DELAY_SEC)  # 5 Seconds pause before new grid cycle
                        continue

                    # 2. Sequential Grid Management
                    ask, bid = await get_current_price(ws)
                    if ask is None:
                        await asyncio.sleep(1)
                        continue

                    # Place initial pending buy/sell orders if none placed
                    if placed_buy_levels == 0 and placed_sell_levels == 0:
                        placed_buy_levels = 1
                        placed_sell_levels = 1
                        await telegram_app.bot.send_message(
                            chat_id=TELEGRAM_CHAT_ID,
                            text=f"🚀 Starting Grid Cycle!\nInitial Buy/Sell Stop levels set at {FIRST_DISTANCE_POINTS} points distance."
                        )

                    await asyncio.sleep(1) # Tick interval

        except Exception as e:
            logging.error(f"WebSocket Error: {e}")
            await asyncio.sleep(5)

# ==========================================
# TELEGRAM CONTROLLER COMMANDS
# ==========================================
async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global is_bot_active
    if update.effective_chat.id != TELEGRAM_CHAT_ID:
        return
    is_bot_active = True
    await update.message.reply_text("🟢 Grid Trading Bot ACTIVATED from your phone!")

async def stop_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global is_bot_active
    if update.effective_chat.id != TELEGRAM_CHAT_ID:
        return
    is_bot_active = False
    await update.message.reply_text("🔴 Grid Trading Bot STOPPED.")

async def status_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != TELEGRAM_CHAT_ID:
        return
    status = "RUNNING 🟢" if is_bot_active else "STOPPED 🔴"
    msg = (
        f"📊 **Bot Status**: {status}\n"
        f"🎯 Target Profit: ${TARGET_PROFIT_USD}\n"
        f"📐 Active Buy Levels: {placed_buy_levels}/{TOTAL_LEVELS}\n"
        f"📐 Active Sell Levels: {placed_sell_levels}/{TOTAL_LEVELS}"
    )
    await update.message.reply_text(msg, parse_mode="Markdown")

# ==========================================
# MAIN EXECUTION
# ==========================================
def main():
    telegram_app = ApplicationBuilder().token(TELEGRAM_BOT_TOKEN).build()

    telegram_app.add_handler(CommandHandler("start", start_command))
    telegram_app.add_handler(CommandHandler("stop", stop_command))
    telegram_app.add_handler(CommandHandler("status", status_command))

    # Run Python Engine & Telegram bot concurrently
    loop = asyncio.get_event_loop()
    loop.create_task(grid_engine(telegram_app))
    
    print("Python Trading Bridge running... Open Telegram on your phone to control.")
    telegram_app.run_polling()

if __name__ == "__main__":
    main()
