import requests
import pandas as pd
import matplotlib.pyplot as plt
import json
import time
from datetime import datetime

# ================================================================
#      CONFIGURATION - change these values to your own
# ================================================================

CONFIG = {
    # Balance & risk settings
    "initial_balance": 100,   # USD
    "max_drawdown": 0.10,     # 10% max drawdown
    "trailing_pct": 0.05,     # 5% trailing stop

    # Signal thresholds
    "prob_yes_min": 12,       # BUY YES if <= 12%
    "prob_yes_max": 88,       # BUY NO if >= 88%

    # Market filters
    "min_volume": 1500,       # minimum 24h volume
    "min_liquidity": 750,     # minimum liquidity

    # Bot settings
    "market_limit": 50,
    "cycle": 3,
    "interval": 60,           # seconds

    # Output files (harus ada ekstensi!)
    "signal_file": "signal.csv",
    "backtest_file": "backtest.csv",
    "log_file": "log.csv",
}

# ================================================================
#   1: FETCH DATA FROM POLYMARKET API
# ================================================================

def fetch_polymarket():
    url = "https://gamma-api.polymarket.com/markets"   # tanpa /v1
    params = {
        "limit": CONFIG["market_limit"],
        "active": "true",
        "closed": "false",
        "order": "volume24hr",      # nama field yang benar
        "ascending": "false",
    }
    try:
        response = requests.get(url, params=params, timeout=10)
        if response.status_code == 200:
            return response.json()
        print(f"   ⚠️ API Error: {response.status_code}")
        return None
    except Exception as e:
        print(f"   ⚠️ Connection Error: {e}")
        return None

# ================================================================
#   2: PROCESS DATA TO DATAFRAME
# ================================================================

def to_float(value):
    try:
        return float(value)
    except (TypeError, ValueError):
        return 0.0

def process_data(data):
    rows = []
    for market in data:
        try:
            prices = market.get("outcomePrices", "[0,0]")
            if isinstance(prices, str):
                prices = json.loads(prices)
            prob_yes = round(float(prices[0]) * 100, 2)
            prob_no = round(float(prices[1]) * 100, 2)
        except Exception:
            continue  # skip market dengan harga rusak (jangan dianggap 0%)

        rows.append({
            "question": market.get("question", "Unknown"),
            "prob_yes": prob_yes,
            "prob_no": prob_no,
            "volume24h": round(to_float(market.get("volume24hr")), 2),
            "liquidity": round(to_float(market.get("liquidity")), 2),
            "end_date": (market.get("endDate") or "")[:10],
        })
    return pd.DataFrame(rows, columns=["question", "prob_yes", "prob_no",
                                       "volume24h", "liquidity", "end_date"])

# ================================================================
#   3: SCANNER - FIND OPPORTUNITIES
# ================================================================

def bot_decision(question, prob_yes, volume24h, liquidity):
    if volume24h < CONFIG["min_volume"]:
        return "SKIP", "Volume too low"
    if liquidity < CONFIG["min_liquidity"]:
        return "SKIP", "Liquidity too low"

    if prob_yes <= CONFIG["prob_yes_min"]:
        return "BUY YES", f"Underdog interesting (prob_yes={prob_yes}%)"
    elif prob_yes >= CONFIG["prob_yes_max"]:
        return "BUY NO", f"NO cheap - YES {prob_yes}%"
    return "SKIP", f"No Alpha signal (prob_yes={prob_yes}%)"

def scan_market(df):
    signals = []
    for _, row in df.iterrows():
        action, reason = bot_decision(
            row["question"], row["prob_yes"],
            row["volume24h"], row["liquidity"]
        )
        if action != "SKIP":
            signals.append({
                "question": row["question"],
                "prob_yes": row["prob_yes"],
                "prob_no": row["prob_no"],
                "volume24h": row["volume24h"],
                "liquidity": row["liquidity"],
                "action": action,
                "reason": reason,
                "end_date": row["end_date"],
                "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            })
    return signals

# ================================================================
#   4: RISK MANAGEMENT
# ================================================================

class RiskManager:
    def __init__(self):
        self.initial_balance = CONFIG["initial_balance"]
        self.balance = CONFIG["initial_balance"]
        self.highest_balance = CONFIG["initial_balance"]
        self.max_drawdown = CONFIG["max_drawdown"]
        self.trailing_pct = CONFIG["trailing_pct"]
        self.active = True
        self.mode = "NORMAL ✅"
        self.loss_streak = 0
        self.trade_history = []

    def update_mode(self):
        drawdown = (self.initial_balance - self.balance) / self.initial_balance
        if drawdown <= 0:
            self.mode = "AGGRESSIVE 🚀"
        elif drawdown <= 0.05:
            self.mode = "NORMAL ✅"
        elif drawdown <= 0.10:
            self.mode = "DEFENSIVE 🛡️"
        elif drawdown <= 0.15:
            self.mode = "RECOVERY 🔄"
        else:
            self.mode = "SURVIVAL ⚠️"

    def calculate_bet(self, side_prob):
        """side_prob = probabilitas (%) sisi yang dibeli."""
        if not self.active:
            return 0
        mode_risk = {
            "AGGRESSIVE 🚀": 0.03,
            "NORMAL ✅": 0.02,
            "DEFENSIVE 🛡️": 0.015,
            "RECOVERY 🔄": 0.01,
            "SURVIVAL ⚠️": 0.005,
        }
        if side_prob <= 10:
            edge_mult = 1.0
        elif side_prob <= 15:
            edge_mult = 0.75
        else:
            edge_mult = 0.5
        risk_pct = mode_risk.get(self.mode, 0.02)
        bet = self.balance * risk_pct * edge_mult
        if self.loss_streak >= 3:
            bet *= 0.5
            print(f"   ⚠️ Loss streak {self.loss_streak}x - halving bet 50%!")
        return round(max(bet, 0.10), 2)  # minimum $0.10

    def check_trailing_stop(self):
        if self.balance > self.highest_balance:
            self.highest_balance = self.balance
            print(f"    🔺 New highest balance: ${self.highest_balance:.2f}")
        trailing_stop = self.highest_balance * (1 - self.trailing_pct)
        initial_drawdown = (self.initial_balance - self.balance) / self.initial_balance
        if self.balance <= trailing_stop:
            self.active = False
            print(f"  ⚠️ TRAILING STOP! Balance: ${self.balance:.2f}")
            return False
        if initial_drawdown >= self.max_drawdown:
            self.active = False
            print(f"  ⚠️ MAX DRAWDOWN reached! Balance: ${self.balance:.2f}")
            return False
        return True

# ================================================================
#   5: VISUALIZER - dashboard
# ================================================================

def show_dashboard(df, signals, rm, log_cycle):
    print("\n" + "=" * 65)
    print("   POLYMARKET TRADER'S WET DREAM v1.0")
    print(f"  {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print("=" * 65)
    print(f"  Total markets monitored : {len(df)}")
    print(f"  Total signals           : {len(signals)}")
    print(f"  Initial Balance         : ${rm.initial_balance:.2f}")
    print(f"  Current Balance         : ${rm.balance:.2f}")
    print(f"  Highest Balance         : ${rm.highest_balance:.2f}")
    print(f"  PnL                     : ${rm.balance - rm.initial_balance:.2f}")
    print(f"  Current Mode            : {rm.mode}")
    print(f"  Active Bot              : {rm.active}")
    print("=" * 65)

    if signals:
        print(f"\n{'MARKET':<45} {'YES%':>6} {'ACTION':>10} {'BET':>8}")
        print("-" * 75)
        for s in signals[-10:]:  # 10 sinyal terakhir
            print(f"{s['question'][:44]:<45} {s['prob_yes']:>5}% "
                  f"{s['action']:>10} ${s.get('bet', 0):>6.2f}")

    if len(log_cycle) > 0 and len(df) > 0:
        fig, axes = plt.subplots(2, 2, figsize=(15, 10))

        # Chart 1 - Balance per cycle
        balance = [CONFIG["initial_balance"]] + [s["balance"] for s in log_cycle]
        axes[0, 0].plot(range(len(balance)), balance,
                        marker="o", color="orange", linewidth=2)
        axes[0, 0].axhline(y=CONFIG["initial_balance"], color="gray",
                           linestyle="--", linewidth=1, label="Initial balance")
        axes[0, 0].axhline(y=rm.highest_balance * (1 - CONFIG["trailing_pct"]),
                           color="red", linestyle="--", linewidth=1,
                           label="Trailing stop")
        axes[0, 0].set_title("Balance per cycle", fontweight="bold")
        axes[0, 0].set_ylabel("Balance (USD)")
        axes[0, 0].legend(fontsize=8)
        axes[0, 0].grid(alpha=0.3)

        # Chart 2 - signals per cycle
        cycle_nums = [s["cycle"] for s in log_cycle]
        signals_count = [s["signals"] for s in log_cycle]
        axes[0, 1].bar(cycle_nums, signals_count, color="#3498db", alpha=0.8)
        axes[0, 1].set_title("Signals found per cycle", fontweight="bold")
        axes[0, 1].set_xlabel("Cycle")
        axes[0, 1].set_ylabel("Number of Signals")
        axes[0, 1].grid(axis="y", alpha=0.3)

        # Chart 3 - Distribution of prob_yes
        axes[1, 0].hist(df["prob_yes"], bins=20,
                        color="#9b59b6", alpha=0.8, edgecolor="white")
        axes[1, 0].axvline(x=CONFIG["prob_yes_min"], color="green",
                           linestyle="--", linewidth=2,
                           label=f"BUY YES (<={CONFIG['prob_yes_min']}%)")
        axes[1, 0].axvline(x=CONFIG["prob_yes_max"], color="red",
                           linestyle="--", linewidth=2,
                           label=f"BUY NO (>={CONFIG['prob_yes_max']}%)")
        axes[1, 0].set_title("Market Probability Distribution", fontweight="bold")
        axes[1, 0].set_xlabel("YES Probability (%)")
        axes[1, 0].set_ylabel("Number of Markets")
        axes[1, 0].legend(fontsize=8)
        axes[1, 0].grid(alpha=0.3)

        # Chart 4 - Volume vs probability
        axes[1, 1].scatter(df["prob_yes"], df["volume24h"],
                           alpha=0.5, color="#1abc9c", s=50)
        if signals:
            df_signals = pd.DataFrame(signals)
            colors = ["green" if a == "BUY YES" else "red"
                      for a in df_signals["action"]]
            axes[1, 1].scatter(df_signals["prob_yes"], df_signals["volume24h"],
                               c=colors, s=150, zorder=5,
                               edgecolors="black", linewidth=0.5)
        axes[1, 1].axvline(x=CONFIG["prob_yes_min"], color="green",
                           linestyle="--", linewidth=1)
        axes[1, 1].axvline(x=CONFIG["prob_yes_max"], color="red",
                           linestyle="--", linewidth=1)
        axes[1, 1].set_title("Volume vs Probability", fontweight="bold")
        axes[1, 1].set_xlabel("YES Probability (%)")
        axes[1, 1].set_ylabel("Volume 24H (USD)")
        axes[1, 1].grid(alpha=0.3)

        plt.suptitle("POLYMARKET TRADER'S WET DREAM v1.0 — Dashboard",
                     fontsize=13, fontweight="bold")
        plt.tight_layout()
        plt.savefig("dashboard.png", dpi=120)
        plt.show()

# ================================================================
#   MAIN - RUN THE BOT
# ================================================================

def running_bot():
    print("=" * 65)
    print("  POLYMARKET TRADER'S WET DREAM v1.0")
    print(f"  {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"  Balance: ${CONFIG['initial_balance']} | "
          f"Thresholds: YES<={CONFIG['prob_yes_min']}%, NO>={CONFIG['prob_yes_max']}% | "
          f"risk: 0.5-3%")
    print("=" * 65)

    rm = RiskManager()
    all_cycle = []
    all_signals = []
    df_last = None

    for i in range(CONFIG["cycle"]):
        if not rm.active:
            print("\n⛔ Bot stopped due to risk management.")
            break

        print(f"\n{'=' * 65}")
        print(f"CYCLE #{i + 1}/{CONFIG['cycle']} - "
              f"{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"Balance: ${rm.balance:.2f} | "
              f"Highest: ${rm.highest_balance:.2f} | "
              f"Trailing Stop: ${rm.highest_balance * (1 - CONFIG['trailing_pct']):.2f}")
        print("=" * 65)

        print("  🔄 Fetching data from Polymarket API...")
        data = fetch_polymarket()
        if not data:
            print("  ❌ Failed to fetch data from Polymarket API.")
            if i < CONFIG["cycle"] - 1:
                time.sleep(CONFIG["interval"])
            continue

        df = process_data(data)
        df_last = df
        rm.update_mode()

        signals_cycle = scan_market(df)

        if signals_cycle:
            print(f"\n⚡ {len(signals_cycle)} SIGNALS!")
            for s in signals_cycle:
                side_prob = s["prob_yes"] if s["action"] == "BUY YES" else s["prob_no"]
                bet = rm.calculate_bet(side_prob)
                s["bet"] = bet
                s["mode"] = rm.mode
                print(f"\n   Market  : {s['question']}")
                print(f"   YES%    : {s['prob_yes']}% | NO%: {s['prob_no']}%")
                print(f"   Action  : {s['action']}")
                print(f"   Bet     : ${bet:.2f} ({rm.mode})")
                print(f"   Reason  : {s['reason']}")
                print(f"   Ends    : {s['end_date']}")
            all_signals.extend(signals_cycle)
        else:
            print(" ⚠️ No signals found this cycle.")

        all_cycle.append({
            "cycle": i + 1,
            "signals": len(signals_cycle),
            "balance": rm.balance,
            "mode": rm.mode,
        })

        if all_signals:
            pd.DataFrame(all_signals).to_csv(CONFIG["signal_file"], index=False)

        rm.check_trailing_stop()

        print(f"\n📊 Cycle #{i + 1}: {len(signals_cycle)} signals  |  "
              f"Balance: ${rm.balance:.2f} | Mode: {rm.mode}")

        if i < CONFIG["cycle"] - 1:
            print(f"\n⏳ Waiting for {CONFIG['interval']} seconds before next cycle...")
            time.sleep(CONFIG["interval"])

    if df_last is not None:
        show_dashboard(df_last, all_signals, rm, all_cycle)

    print("\n" + "=" * 65)
    print(" POLYMARKET TRADER'S WET DREAM v1.0 - SUMMARY")
    print("=" * 65)
    print(f" Total cycle    : {len(all_cycle)}")
    print(f" Total signals  : {len(all_signals)}")
    print(f" Initial Balance: ${rm.initial_balance:.2f}")
    print(f" Final Balance  : ${rm.balance:.2f}")
    print(f" P&L            : ${rm.balance - rm.initial_balance:.2f}")
    print(f" Bot active     : {rm.active}")
    if all_signals:
        print(f"\n✅ Data saved to {CONFIG['signal_file']}")

    return all_cycle, all_signals


if __name__ == "__main__":
    cycles, signals = running_bot() 
