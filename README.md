To enable trading via **toxi_solana_bot** in Telegram, add notifications for buys/sells, and merge all the code into one cohesive bot, we need to:

1. **Integrate Telegram Bot**: Use the `python-telegram-bot` library to interact with the toxi_solana_bot.
2. **Enable Trading**: Send buy/sell commands to the toxi_solana_bot.
3. **Add Notifications**: Notify the user via Telegram when a buy/sell occurs.
4. **Merge All Code**: Combine the GMGN API, SolSniffer, RugCheck, and trading logic into one script.
5. **Write a Launch Guide**: Provide step-by-step instructions for setting up and running the bot.

---

### **1. Install Required Libraries**
Install the necessary Python libraries:

```bash
pip install requests pandas python-telegram-bot schedule sqlite3
```

---

### **2. Telegram Bot Integration**
Use the `python-telegram-bot` library to interact with the toxi_solana_bot.

```python
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext

# Replace with your Telegram bot token
TELEGRAM_BOT_TOKEN = "your_telegram_bot_token"
TELEGRAM_CHAT_ID = "your_chat_id"  # Your Telegram chat ID

def send_telegram_message(message):
    updater = Updater(TELEGRAM_BOT_TOKEN)
    updater.bot.send_message(chat_id=TELEGRAM_CHAT_ID, text=message)

def buy_command(update: Update, context: CallbackContext):
    token_address = context.args[0]
    # Send buy command to toxi_solana_bot
    send_telegram_message(f"/buy {token_address}")
    update.message.reply_text(f"Buy command sent for token: {token_address}")

def sell_command(update: Update, context: CallbackContext):
    token_address = context.args[0]
    # Send sell command to toxi_solana_bot
    send_telegram_message(f"/sell {token_address}")
    update.message.reply_text(f"Sell command sent for token: {token_address}")

# Set up Telegram bot handlers
updater = Updater(TELEGRAM_BOT_TOKEN)
updater.dispatcher.add_handler(CommandHandler("buy", buy_command))
updater.dispatcher.add_handler(CommandHandler("sell", sell_command))
updater.start_polling()
```

---

### **3. Trading Logic**
Add functions to execute buy/sell commands based on analysis.

```python
def execute_buy(coin_id):
    # Notify user
    send_telegram_message(f"🚀 Buying {coin_id} 🚀")
    # Send buy command to toxi_solana_bot
    send_telegram_message(f"/buy {coin_id}")

def execute_sell(coin_id):
    # Notify user
    send_telegram_message(f"📉 Selling {coin_id} 📉")
    # Send sell command to toxi_solana_bot
    send_telegram_message(f"/sell {coin_id}")
```

---

### **4. Merge All Code**
Combine the GMGN API, SolSniffer, RugCheck, and trading logic into one script.

```python
import requests
import json
import sqlite3
import pandas as pd
import schedule
import time
import logging
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext

# Load configuration
def load_config():
    with open("config.json", "r") as f:
        return json.load(f)

config = load_config()

# Initialize clients
class GMGNClient:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.gmgn.ai/v1"

    def get_coin_data(self, coin_id):
        headers = {"Authorization": f"Bearer {self.api_key}"}
        response = requests.get(f"{self.base_url}/coins/{coin_id}", headers=headers)
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"Failed to fetch data: {response.status_code}")

class SolSnifferClient:
    def __init__(self, api_key=None):
        self.base_url = "https://api.solsniffer.com"
        self.api_key = api_key

    def get_token_score(self, token_address):
        headers = {"Authorization": f"Bearer {self.api_key}"} if self.api_key else {}
        response = requests.get(f"{self.base_url}/token/{token_address}/score", headers=headers)
        if response.status_code == 200:
            return response.json().get("score")
        else:
            raise Exception(f"Failed to fetch score for token {token_address}: {response.status_code}")

class RugCheckClient:
    def __init__(self, api_key=None):
        self.base_url = "https://api.rugcheck.xyz"
        self.api_key = api_key

    def get_contract_data(self, contract_address):
        headers = {"Authorization": f"Bearer {self.api_key}"} if self.api_key else {}
        response = requests.get(f"{self.base_url}/contract/{contract_address}", headers=headers)
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"Failed to fetch data for contract {contract_address}: {response.status_code}")

# Initialize database
def setup_database():
    conn = sqlite3.connect("crypto_data.db")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS coin_data (
            id TEXT PRIMARY KEY,
            name TEXT,
            price REAL,
            volume REAL,
            market_cap REAL,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS flagged_tokens (
            id TEXT PRIMARY KEY,
            name TEXT,
            score REAL,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    return conn

# Save coin data
def save_coin_data(conn, coin_data, config):
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO coin_data (id, name, price, volume, market_cap)
        VALUES (?, ?, ?, ?, ?)
    """, (coin_data["id"], coin_data["name"], coin_data["price"], coin_data["volume"], coin_data["market_cap"]))
    conn.commit()

# Analyze pumps and rugs
def analyze_pumps(conn, config):
    query = "SELECT * FROM coin_data"
    df = pd.read_sql(query, conn)
    df["price_change"] = df.groupby("id")["price"].pct_change() * 100
    filtered_df = df[
        (df["volume"] >= config["filters"]["min_volume"]) &
        (df["market_cap"] <= config["filters"]["max_market_cap"]) &
        (abs(df["price_change"]) >= config["filters"]["price_change_threshold"])
    ]
    pumped_coins = filtered_df[filtered_df["price_change"] > 0]
    rugged_coins = filtered_df[filtered_df["price_change"] < 0]
    return pumped_coins, rugged_coins

# Flag unsafe tokens
def flag_unsafe_tokens(conn, sol_sniffer_client, config):
    cursor = conn.cursor()
    cursor.execute("SELECT id, name FROM coin_data")
    coins = cursor.fetchall()
    flagged_tokens = []
    for coin_id, coin_name in coins:
        try:
            score = sol_sniffer_client.get_token_score(coin_id)
            if score is not None and score < config["filters"]["min_safety_score"]:
                flagged_tokens.append((coin_id, coin_name, score))
                logging.info(f"Flagged token: {coin_name} (ID: {coin_id}, Score: {score})")
        except Exception as e:
            logging.error(f"Error fetching score for token {coin_id}: {e}")
    return flagged_tokens

# Update blacklist
def update_blacklist(coin_id):
    with open("config.json", "r") as f:
        config = json.load(f)
    if coin_id not in config["coin_blacklist"]:
        config["coin_blacklist"].append(coin_id)
    with open("config.json", "w") as f:
        json.dump(config, f, indent=4)
    print(f"Added {coin_id} to the blacklist.")

# Execute buy/sell commands
def execute_buy(coin_id):
    send_telegram_message(f"🚀 Buying {coin_id} 🚀")
    send_telegram_message(f"/buy {coin_id}")

def execute_sell(coin_id):
    send_telegram_message(f"📉 Selling {coin_id} 📉")
    send_telegram_message(f"/sell {coin_id}")

# Main job
def job():
    print("Fetching and analyzing data...")
    all_coins = get_all_coins(client)
    for coin in all_coins:
        coin_id = coin["id"]
        try:
            contract_data = rug_check_client.get_contract_data(coin_id)
            if not is_good_contract(contract_data):
                logging.info(f"Skipping non-Good contract: {coin_id}")
                continue
            if has_bundled_supply(contract_data):
                logging.warning(f"Flagging bundled supply token: {coin_id}")
                update_blacklist(coin_id)
                continue
            coin_data = client.get_coin_data(coin_id)
            save_coin_data(conn, coin_data, config)
        except Exception as e:
            logging.error(f"Error processing coin {coin_id}: {e}")
    pumped_coins, rugged_coins = analyze_pumps(conn, config)
    logging.info(f"Pumped Coins: {pumped_coins}")
    logging.info(f"Rugged Coins: {rugged_coins}")

# Schedule the job
schedule.every().hour.do(job)

# Run the bot
if __name__ == "__main__":
    logging.basicConfig(filename="bot.log", level=logging.INFO)
    conn = setup_database()
    client = GMGNClient(api_key="your_gmgn_api_key")
    sol_sniffer_client = SolSnifferClient(api_key="your_sol_sniffer_api_key")
    rug_check_client = RugCheckClient(api_key="your_rugcheck_api_key")
    updater = Updater(TELEGRAM_BOT_TOKEN)
    updater.dispatcher.add_handler(CommandHandler("buy", buy_command))
    updater.dispatcher.add_handler(CommandHandler("sell", sell_command))
    updater.start_polling()
    while True:
        schedule.run_pending()
        time.sleep(1)
```

---

### **5. Launch Guide**

#### **Step 1: Set Up Environment**
1. Install Python 3.8+.
2. Install required libraries:
   ```bash
   pip install requests pandas python-telegram-bot schedule sqlite3
   ```

#### **Step 2: Configure the Bot**
1. Create a `config.json` file:
   ```json
   {
     "filters": {
       "min_volume": 1000000,
       "max_market_cap": 100000000,
       "price_change_threshold": 50,
       "min_safety_score": 80
     },
     "coin_blacklist": [],
     "dev_blacklist": []
   }
   ```
2. Replace API keys in the script with your actual keys:
   - GMGN API key.
   - SolSniffer API key.
   - RugCheck API key.
   - Telegram bot token.

#### **Step 3: Run the Bot**
1. Save the script as `crypto_bot.py`.
2. Run the bot:
   ```bash
   python crypto_bot.py
   ```

#### **Step 4: Interact via Telegram**
- Use `/buy TOKEN_ADDRESS` to buy a token.
- Use `/sell TOKEN_ADDRESS` to sell a token.

---

This guide provides a complete solution for launching the bot. Let me know if you need further assistance!
