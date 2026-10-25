# harsh-trading-alerts

const MARKET_API = "https://data-api.binance.vision";

const SYMBOLS = {
  BTC: "BTCUSDT",
  BTCUSDT: "BTCUSDT",
  ETH: "ETHUSDT",
  ETHUSDT: "ETHUSDT",
  SOL: "SOLUSDT",
  SOLUSDT: "SOLUSDT",
  BNB: "BNBUSDT",
  BNBUSDT: "BNBUSDT",
  XRP: "XRPUSDT",
  XRPUSDT: "XRPUSDT"
};

function resolveSymbol(input) {
  const key = (input || "BTC").toUpperCase();
  return SYMBOLS[key] || null;
}

function formatPrice(value) {
  return Number(value).toLocaleString("en-US", {
    maximumFractionDigits: 8
  });
}

async function telegramSend(env, chatId, text) {
  const response = await fetch(
    `https://api.telegram.org/bot${env.BOT_TOKEN}/sendMessage`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        chat_id: chatId,
        text
      })
    }
  );

  if (!response.ok) {
    throw new Error("Telegram sendMessage failed");
  }
}

async function marketPrice(symbol) {
  const response = await fetch(
    `${MARKET_API}/api/v3/ticker/price?symbol=${symbol}`
  );

  if (!response.ok) {
    throw new Error("Market price request failed");
  }

  const data = await response.json();
  return Number(data.price);
}

function emaSeries(values, period) {
  const output = Array(values.length).fill(null);

  if (values.length < period) return output;

  let sum = 0;
  for (let i = 0; i < period; i++) {
    sum += values[i];
  }

  let current = sum / period;
  output[period - 1] = current;

  const multiplier = 2 / (period + 1);

  for (let i = period; i < values.length; i++) {
    current =
      values[i] * multiplier +
      current * (1 - multiplier);

    output[i] = current;
  }

  return output;
}

function calculateRSI(values, period = 14) {
  if (values.length <= period) return null;

  let gains = 0;
  let losses = 0;

  for (let i = 1; i <= period; i++) {
    const change = values[i] - values[i - 1];
    gains += Math.max(change, 0);
    losses += Math.max(-change, 0);
  }

  let averageGain = gains / period;
  let averageLoss = losses / period;

  for (let i = period + 1; i < values.length; i++) {
    const change = values[i] - values[i - 1];

    averageGain =
      (averageGain * (period - 1) + Math.max(change, 0)) /
      period;

    averageLoss =
      (averageLoss * (period - 1) + Math.max(-change, 0)) /
      period;
  }

  if (averageLoss === 0) return averageGain === 0 ? 50 : 100;

  const relativeStrength = averageGain / averageLoss;
  return 100 - 100 / (1 + relativeStrength);
}

async function getSignal(symbol) {
  const response = await fetch(
    `${MARKET_API}/api/v3/klines?symbol=${symbol}&interval=5m&limit=100`
  );

  if (!response.ok) {
    throw new Error("Candlestick request failed");
  }

  const candles = await response.json();

  // Ignore the latest candle because it may still be forming.
  const closedCandles = candles.slice(0, -1);
  const closes = closedCandles.map(candle => Number(candle[4]));

  if (closes.length < 30) {
    throw new Error("Not enough closed candles");
  }

  const fastEMA = emaSeries(closes, 9);
  const slowEMA = emaSeries(closes, 21);
  const last = closes.length - 1;
  const previous = last - 1;
  const rsi = calculateRSI(closes);

  const crossedUp =
    fastEMA[previous] <= slowEMA[previous] &&
    fastEMA[last] > slowEMA[last];

  const crossedDown =
    fastEMA[previous] >= slowEMA[previous] &&
    fastEMA[last] < slowEMA[last];

  let result = "NO FRESH SETUP";

  if (crossedUp && rsi >= 50 && rsi < 70) {
    result = "BULLISH CROSSOVER ALERT";
  } else if (crossedDown && rsi > 30 && rsi <= 50) {
    result = "BEARISH CROSSOVER ALERT";
  }

  const trend =
    fastEMA[last] > slowEMA[last]
      ? "Bullish EMA trend"
      : fastEMA[last] < slowEMA[last]
        ? "Bearish EMA trend"
        : "Neutral EMA trend";

  return [
    `📊 ${symbol} — 5-minute analysis`,
    `Last closed price: ${formatPrice(closes[last])} USDT`,
    `EMA 9: ${formatPrice(fastEMA[last])}`,
    `EMA 21: ${formatPrice(slowEMA[last])}`,
    `RSI (14): ${rsi.toFixed(2)}`,
    `Trend: ${trend}`,
    `Signal: ${result}`,
    "",
    "Rules: EMA 9/21 crossover with an RSI filter.",
    "A signal is an indicator condition, not a prediction.",
    "Test with paper trading; no orders are placed."
  ].join("\n");
}

async function handleUpdate(update, env) {
  const message = update.message;

  if (!message || !message.text || !message.chat) return;

  const chatId = message.chat.id;
  const parts = message.text.trim().split(/\s+/);
  const command = parts[0].split("@")[0].toLowerCase();
  const argument = parts[1] || "BTC";

  try {
    if (command === "/start") {
      await telegramSend(env, chatId, [
        "Welcome to Harsh Trading Alerts! 📈",
        "",
        "/help — Show commands",
        "/price BTC — Bitcoin price",
        "/price ETH — Ethereum price",
        "/price SOL — Solana price",
        "/signal BTC — Bitcoin strategy analysis",
        "/signal ETH — Ethereum strategy analysis",
        "",
        "Market data is public. This bot does not place trades."
      ].join("\n"));

    } else if (command === "/help") {
      await telegramSend(env, chatId, [
        "Available commands:",
        "/start",
        "/help",
        "/price BTC",
        "/price ETH",
        "/price SOL",
        "/price BNB",
        "/price XRP",
        "/signal BTC",
        "/signal ETH",
        "/signal SOL",
        "/signal BNB",
        "/signal XRP"
      ].join("\n"));

    } else if (command === "/price") {
      const symbol = resolveSymbol(argument);

      if (!symbol) {
        await telegramSend(
          env,
          chatId,
          "Unsupported asset. Try BTC, ETH, SOL, BNB or XRP."
        );
        return;
      }

      const price = await marketPrice(symbol);

      await telegramSend(
        env,
        chatId,
        `💰 ${symbol}\nPrice: ${formatPrice(price)} USDT\nSource: Binance public market data`
      );

    } else if (command === "/signal") {
      const symbol = resolveSymbol(argument);

      if (!symbol) {
        await telegramSend(
          env,
          chatId,
          "Unsupported asset. Try BTC, ETH, SOL, BNB or XRP."
        );
        return;
      }

      const signal = await getSignal(symbol);
      await telegramSend(env, chatId, signal);

    } else {
      await telegramSend(
        env,
        chatId,
        "Unknown command. Send /help to see the available commands."
      );
    }
  } catch (error) {
    console.error("Command processing failed:", error.message);

    await telegramSend(
      env,
      chatId,
      "Sorry, I couldn't retrieve the market data just now. Please try again shortly."
    );
  }
}

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    if (request.method === "GET" && url.pathname === "/") {
      return new Response("Harsh Trading Alerts worker is online.");
    }

    if (request.method !== "POST" || url.pathname !== "/telegram") {
      return new Response("Not found", { status: 404 });
    }

    if (!env.BOT_TOKEN || !env.WEBHOOK_SECRET) {
      return new Response("Worker secrets are not configured.", {
        status: 500
      });
    }

    if (
      request.headers.get("X-Telegram-Bot-Api-Secret-Token") !==
      env.WEBHOOK_SECRET
    ) {
      return new Response("Forbidden", { status: 403 });
    }

    let update;

    try {
      update = await request.json();
    } catch {
      return new Response("Invalid JSON", { status: 400 });
    }

    ctx.waitUntil(
      handleUpdate(update, env).catch(error => {
        console.error("Telegram update failed:", error.message);
      })
    );

    return new Response("OK");
  }
};
