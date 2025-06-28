from flask import Flask
import yfinance as yf
import requests
from datetime import datetime, timedelta
import threading
import time

app = Flask(__name__)

TELE_TOKEN  = "7506855523:AAEoD-2xo2H82pSLsR9MYJQ7z5Jefpw9POc"
TELE_CHAT   = "7708999134"
PAIRS       = ["EURUSD=X","GBPUSD=X","USDJPY=X","USDCHF=X","USDCAD=X"]
GAPS        = [5,10,8,4,3,30]
gap_idx     = 0
next_allowed= datetime.utcnow() + timedelta(hours=8)
sent_signals= set()

def market_open():
    now = datetime.utcnow() + timedelta(hours=8)
    wd, hr = now.weekday(), now.hour
    return not ((wd==5 and hr>=5) or wd==6 or (wd==0 and hr<5))

def get_signal(ticker):
    df = yf.download(ticker, period="2m", interval="1m", progress=False)
    if len(df)<2: return None
    prev, last = df['Close'][-2], df['Close'][-1]
    if abs(last-prev)<0.0001: return None
    dt_last = df.index[-1].to_pydatetime() + timedelta(hours=8)
    dt_next = dt_last + timedelta(minutes=1)
    key = f"{ticker}-{dt_next.strftime('%H:%M')}"
    if key in sent_signals: return None
    return {
        "pair": ticker.replace("=X",""),
        "type": "CALL" if last>prev else "PUT",
        "next_time": dt_next.strftime("%H:%M"),
        "key": key,
        "arrow": ("🟩" if last>prev else "🟥")*8
    }

def send_telegram(sig):
    txt = (
        f"✿ RD SIGNAL PRO ✿\n"
        f"----------------------\n"
        f"Next → {sig['next_time']}\n"
        f"Pair → {sig['pair']}\n"
        f"Type → {sig['type']}\n"
        f"{sig['arrow']}"
    )
    requests.post(
        f"https://api.telegram.org/bot{TELE_TOKEN}/sendMessage",
        data={"chat_id": TELE_CHAT, "text": txt}, timeout=10
    )

def run_signals():
    global next_allowed, gap_idx
    while True:
        now = datetime.utcnow() + timedelta(hours=8)
        if not market_open():
            time.sleep(600)
            continue
        if now < next_allowed:
            time.sleep(min((next_allowed-now).total_seconds(), 60))
            continue
        for tk in PAIRS:
            sig = get_signal(tk)
            if sig:
                send_telegram(sig)
                sent_signals.add(sig["key"])
                next_allowed = now + timedelta(minutes=GAPS[gap_idx % len(GAPS)])
                gap_idx += 1
                break
        else:
            time.sleep(60)

@app.route('/')
def home():
    return "✿ RD SIGNAL PRO ✿ → Running ✅"

if __name__ == '__main__':
    threading.Thread(target=run_signals, daemon=True).start()
    app.run(host='0.0.0.0', port=8000)
    
