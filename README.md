
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Trading HackBot</title>
<style>
*{box-sizing:border-box}
body{
  margin:0;background:#08111f;color:#eef4ff;
  font-family:Arial,sans-serif;padding:16px
}
header{text-align:center;padding:16px 0}
h1{color:#42e8bd;margin-bottom:6px}
p{color:#aab8ce}
.card{
  background:#111e30;border:1px solid #263951;
  padding:16px;border-radius:14px;margin:12px 0
}
label{display:block;margin:12px 0 6px}
input,select,button{
  width:100%;padding:12px;border-radius:8px;
  border:1px solid #344760;background:#091525;
  color:white;font-size:16px
}
button{background:#16b991;border:0;color:#04150f;
  font-weight:bold;margin-top:12px;cursor:pointer}
button.secondary{background:#253851;color:white}
.signal{text-align:center;font-size:28px;font-weight:bold;
  padding:18px;color:#ffd166}
.value{font-size:23px;font-weight:bold}
.small{font-size:13px;line-height:1.5}
.green{color:#42e8bd}.red{color:#ff7777}
</style>
</head>
<body>
<header>
  <h1>⚡ Trading HackBot</h1>
  <p>Forex market analysis and demo trading</p>
</header>

<div class="card">
  <p>MARKET PRICE</p>
  <div class="value" id="price">Not connected</div>
  <p id="updated">Connect a market-data provider to begin.</p>

  <label for="pair">Currency pair</label>
  <select id="pair">
    <option value="EUR/USD">EUR/USD</option>
    <option value="GBP/USD">GBP/USD</option>
    <option value="USD/JPY">USD/JPY</option>
    <option value="AUD/USD">AUD/USD</option>
    <option value="USD/CAD">USD/CAD</option>
  </select>

  <label for="apiKey">Twelve Data API key</label>
  <input id="apiKey" type="password"
    placeholder="Enter your API key">

  <button onclick="connectMarket()">Connect market data</button>
  <button class="secondary" onclick="analyze()">
    Analyze market
  </button>
</div>

<div class="card">
  <p>BOT SIGNAL</p>
  <div class="signal" id="signal">WAIT</div>
  <p id="reason">Connect market data to analyze prices.</p>
  <p>Strategy: 9-period vs 21-period EMA</p>
  <p>Signal refresh: every 30 seconds</p>
  <p class="small">
    Signals are estimates, not guaranteed outcomes.
    A 1-minute candle is not a 30-second candle.
  </p>
</div>

<div class="card">
  <p>DEMO ACCOUNT</p>
  <div class="value" id="balance">$1,000.00</div>
  <label for="stake">Practice stake ($)</label>
  <input id="stake" type="number" value="10" min="1">
  <button onclick="demoTrade('BUY')">Demo BUY</button>
  <button class="secondary" onclick="demoTrade('SELL')">
    Demo SELL
  </button>
  <p id="tradeStatus" class="small">
    Demo only. No real orders are placed.
  </p>
  <button class="secondary" onclick="resetDemo()">
    Reset demo balance
  </button>
</div>

<script>
let candles = [];
let currentPrice = null;
let balance = 1000;
let timer = null;

function ema(values, period) {
  if (values.length < period) return null;
  const k = 2 / (period + 1);
  let result = values.slice(0, period)
    .reduce((a, b) => a + b, 0) / period;

  for (let i = period; i < values.length; i++) {
    result = values[i] * k + result * (1 - k);
  }
  return result;
}

async function connectMarket() {
  const key = document.getElementById("apiKey").value.trim();
  const pair = document.getElementById("pair").value;

  if (!key) {
    alert("Enter your market-data API key first.");
    return;
  }

  if (timer) clearInterval(timer);

  async function refresh() {
    try {
      const url =
        "https://api.twelvedata.com/time_series" +
        "?symbol=" + encodeURIComponent(pair) +
        "&interval=1min&outputsize=50&apikey=" +
        encodeURIComponent(key);

      const response = await fetch(url);
      const data = await response.json();

      if (!response.ok || !data.values) {
        throw new Error(data.message || "No candle data returned.");
      }

      candles = data.values.slice().reverse()
        .map(c => Number(c.close))
        .filter(n => Number.isFinite(n));

      if (candles.length < 21) {
        throw new Error("At least 21 valid candles are needed.");
      }

      currentPrice = candles[candles.length - 1];
      document.getElementById("price").textContent =
        currentPrice.toFixed(5);
      document.getElementById("updated").textContent =
        "Latest available 1-minute candle: " +
        data.values[0].datetime;

      analyze();
    } catch (err) {
      document.getElementById("signal").textContent = "WAIT";
      document.getElementById("reason").textContent =
        "Data error: " + err.message;
    }
  }

  await refresh();
  timer = setInterval(refresh, 30000);
}

function analyze() {
  if (candles.length < 21) {
    document.getElementById("signal").textContent = "WAIT";
    document.getElementById("reason").textContent =
      "Connect a valid market-data feed first.";
    return;
  }

  const fast = ema(candles, 9);
  const slow = ema(candles, 21);
  const previousFast = ema(candles.slice(0, -1), 9);
  const previousSlow = ema(candles.slice(0, -1), 21);

  let signal = "WAIT";
  let reason = "Trend is unclear. Avoid forcing a trade.";

  if (fast > slow && previousFast >= previousSlow) {
    signal = "BUY";
    reason = "Short EMA is above long EMA. Bullish trend estimate.";
  } else if (fast < slow && previousFast <= previousSlow) {
    signal = "SELL";
    reason = "Short EMA is below long EMA. Bearish trend estimate.";
  }

  document.getElementById("signal").textContent = signal;
  document.getElementById("signal").style.color =
    signal === "BUY" ? "#42e8bd" :
    signal === "SELL" ? "#ff7777" : "#ffd166";

  document.getElementById("reason").textContent = reason;
}

function demoTrade(side) {
  const stake = Number(document.getElementById("stake").value);

  if (!Number.isFinite(stake) || stake <= 0 || stake > balance) {
    alert("Enter a valid stake within your demo balance.");
    return;
  }

  balance -= stake;
  document.getElementById("balance").textContent =
    "$" + balance.toFixed(2);

  document.getElementById("tradeStatus").textContent =
    "Demo " + side + " recorded at " +
    (currentPrice === null ? "no available market price" :
    currentPrice.toString()) +
    ". Stake reserved: $" + stake.toFixed(2) +
    ". No real order or profit is generated.";
}

function resetDemo() {
  balance = 1000;
  document.getElementById("balance").textContent = "$1,000.00";
  document.getElementById("tradeStatus").textContent =
    "Demo balance reset. No real funds involved.";
}
</script>
</body>
</html>
