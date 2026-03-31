"""
Momentum Paper Trader — No Brokerage Required
==============================================
Tracks a simulated $100K portfolio using real live market prices.
State is saved in portfolio.json and committed back to the repo
after every run, so nothing is lost between sessions.

Run manually:  python momentum_trader.py
Force rebalance: python momentum_trader.py --rebalance
GitHub Actions: python momentum_trader.py --run-once
"""

import json, os, sys, datetime
import yfinance as yf
import pandas as pd

# ── CONFIG ────────────────────────────────────────────────────────────────────
STARTING_CASH     = 100_000
N_STOCKS          = 15
MOMENTUM_LOOKBACK = 252
SKIP_RECENT       = 21
MIN_PRICE         = 5.0
PORTFOLIO_FILE    = "portfolio.json"
LOG_FILE          = "trade_log.json"

QQQ_UNIVERSE = [
    "MSFT","AAPL","NVDA","AMZN","META","TSLA","GOOGL","GOOG",
    "AVGO","COST","NFLX","AMD","ADBE","QCOM","PEP","CSCO",
    "TMUS","INTU","AMAT","AMGN","TXN","ISRG","MU","BKNG",
    "LRCX","KLAC","REGN","ADI","PANW","CDNS","CRWD","MELI",
    "SNPS","ASML","MDLZ","CTAS","ORLY","MNST","PYPL","ADP",
    "MAR","ABNB","FTNT","NXPI","MCHP","WDAY","PAYX","CPRT","CEG","ROST",
]

# ── STATE ─────────────────────────────────────────────────────────────────────
def load_portfolio():
    if os.path.exists(PORTFOLIO_FILE):
        with open(PORTFOLIO_FILE) as f: return json.load(f)
    return {"cash": STARTING_CASH, "holdings": {}, "cost_basis": {},
            "created": datetime.date.today().isoformat(), "last_rebalance": None}

def save_portfolio(p):
    with open(PORTFOLIO_FILE, "w") as f: json.dump(p, f, indent=2)

def load_log():
    if os.path.exists(LOG_FILE):
        with open(LOG_FILE) as f: return json.load(f)
    return []

def save_log(l):
    with open(LOG_FILE, "w") as f: json.dump(l, f, indent=2)

# ── PRICES ────────────────────────────────────────────────────────────────────
def get_prices(tickers):
    data = yf.download(tickers, period="5d", auto_adjust=True, progress=False)["Close"]
    if isinstance(data, pd.Series): data = data.to_frame(tickers[0])
    return {t: float(data[t].dropna().iloc[-1]) for t in tickers
            if t in data.columns and len(data[t].dropna()) > 0}

def get_momentum_scores():
    print(f"Scoring {len(QQQ_UNIVERSE)} stocks by 12-1 momentum...")
    end   = datetime.date.today()
    start = end - datetime.timedelta(days=420)
    data  = yf.download(QQQ_UNIVERSE, start=start, end=end,
                        auto_adjust=True, progress=False)["Close"]
    scores = {}
    for t in QQQ_UNIVERSE:
        if t not in data.columns: continue
        p = data[t].dropna()
        if len(p) < MOMENTUM_LOOKBACK or p.iloc[-1] < MIN_PRICE: continue
        scores[t] = p.iloc[-SKIP_RECENT] / p.iloc[-MOMENTUM_LOOKBACK] - 1
    return pd.Series(scores).sort_values(ascending=False)

# ── PORTFOLIO MATH ────────────────────────────────────────────────────────────
def portfolio_snapshot(portfolio, prices):
    positions, stock_val = {}, 0
    for t, sh in portfolio["holdings"].items():
        if t not in prices: continue
        price = prices[t]
        val   = sh * price
        cost  = portfolio["cost_basis"].get(t, price)
        positions[t] = {"shares": round(sh,4), "price": round(price,2),
                        "value": round(val,2), "pnl_pct": round((price/cost-1)*100,2)}
        stock_val += val
    total = portfolio["cash"] + stock_val
    return {"positions": positions, "cash": round(portfolio["cash"],2),
            "total": round(total,2), "ret_pct": round((total/STARTING_CASH-1)*100,2)}

# ── TRADES ────────────────────────────────────────────────────────────────────
def sell(portfolio, ticker, prices, log):
    sh = portfolio["holdings"].get(ticker, 0)
    if sh <= 0 or ticker not in prices: return portfolio
    price    = prices[ticker]
    proceeds = sh * price
    portfolio["cash"] += proceeds
    del portfolio["holdings"][ticker]
    del portfolio["cost_basis"][ticker]
    log.append({"date": str(datetime.date.today()), "action": "SELL",
                "ticker": ticker, "shares": round(sh,4),
                "price": round(price,2), "proceeds": round(proceeds,2)})
    print(f"  SELL  {sh:.2f}sh {ticker} @ ${price:.2f} = ${proceeds:,.0f}")
    return portfolio

def buy(portfolio, ticker, dollars, prices, log):
    if ticker not in prices: return portfolio
    price  = prices[ticker]
    shares = dollars / price
    portfolio["cash"] -= dollars
    portfolio["holdings"][ticker]   = portfolio["holdings"].get(ticker, 0) + shares
    portfolio["cost_basis"][ticker] = price
    log.append({"date": str(datetime.date.today()), "action": "BUY",
                "ticker": ticker, "shares": round(shares,4),
                "price": round(price,2), "cost": round(dollars,2)})
    print(f"  BUY   {shares:.2f}sh {ticker} @ ${price:.2f} = ${dollars:,.0f}")
    return portfolio

# ── REBALANCE ─────────────────────────────────────────────────────────────────
def already_rebalanced_this_month(portfolio):
    last = portfolio.get("last_rebalance")
    if not last: return False
    d = datetime.date.fromisoformat(last)
    t = datetime.date.today()
    return d.month == t.month and d.year == t.year

def is_rebalance_day():
    return datetime.date.today().day <= 3

def rebalance(portfolio, log):
    today = datetime.date.today()
    print(f"\n{'='*52}\n  MONTHLY REBALANCE — {today}\n{'='*52}")

    scores = get_momentum_scores()
    if scores.empty:
        print("No momentum data, aborting."); return portfolio

    picks       = scores.head(N_STOCKS)
    new_tickers = set(picks.index)

    print(f"\nTop {N_STOCKS} picks:")
    for i,(t,s) in enumerate(picks.items(),1):
        print(f"  {i:2}. {t:<6}  {s*100:+.1f}%")

    all_tickers = list(new_tickers | set(portfolio["holdings"].keys()))
    prices      = get_prices(all_tickers)

    snap = portfolio_snapshot(portfolio, prices)
    print(f"\nBefore: ${snap['total']:,.0f}  ({snap['ret_pct']:+.1f}%)")

    # Sell exits
    exits = [t for t in list(portfolio["holdings"]) if t not in new_tickers]
    if exits:
        print(f"\nSelling {len(exits)} exits:")
        for t in exits: portfolio = sell(portfolio, t, prices, log)

    # Buy picks — equal weight
    snap2     = portfolio_snapshot(portfolio, prices)
    per_stock = snap2["total"] / N_STOCKS
    print(f"\nBuying {N_STOCKS} picks (${per_stock:,.0f} each):")
    for t in picks.index: portfolio = buy(portfolio, t, per_stock, prices, log)

    portfolio["last_rebalance"] = str(today)
    snap3 = portfolio_snapshot(portfolio, prices)
    print(f"\nAfter:  ${snap3['total']:,.0f}  ({snap3['ret_pct']:+.1f}%)")
    return portfolio

# ── STATUS ────────────────────────────────────────────────────────────────────
def status(portfolio, log):
    today = datetime.date.today()
    print(f"\n{'='*52}\n  DAILY STATUS — {today}\n{'='*52}")
    if not portfolio["holdings"]:
        print("No positions yet — waiting for first rebalance (day 1-3 of month).")
        return
    prices = get_prices(list(portfolio["holdings"].keys()))
    snap   = portfolio_snapshot(portfolio, prices)
    print(f"\nTotal Value:  ${snap['total']:>12,.0f}")
    print(f"Cash:         ${snap['cash']:>12,.0f}")
    print(f"Total Return: {snap['ret_pct']:>+11.2f}%\n")
    print("Positions:")
    for t, pos in sorted(snap["positions"].items(), key=lambda x: x[1]["value"], reverse=True):
        arrow = "▲" if pos["pnl_pct"] >= 0 else "▼"
        print(f"  {t:<6}  {pos['shares']:>8.2f}sh  ${pos['value']:>9,.0f}  "
              f"{arrow} {pos['pnl_pct']:>+6.1f}%")
    log.append({"date": str(today), "type": "snapshot",
                "total": snap["total"], "ret_pct": snap["ret_pct"]})

# ── MAIN ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    portfolio = load_portfolio()
    log       = load_log()
    args      = sys.argv[1:]

    if "--rebalance" in args:
        portfolio = rebalance(portfolio, log)
    elif "--run-once" in args:
        # GitHub Actions mode
        if is_rebalance_day() and not already_rebalanced_this_month(portfolio):
            portfolio = rebalance(portfolio, log)
        else:
            status(portfolio, log)
    else:
        status(portfolio, log)
        print("\nTip: use --rebalance to force a rebalance now.")

    save_portfolio(portfolio)
    save_log(log)
    print("\nDone.")
