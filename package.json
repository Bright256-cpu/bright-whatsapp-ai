import makeWASocket, { useMultiFileAuthState, DisconnectReason } from "@whiskeysockets/baileys";
import http from "http";
import QRCode from "qrcode";

const conversations = new Map();
let latestQR = null;
let isOnline = false;

async function askAI(message, jid) {
  const history = conversations.get(jid) || [];
  history.push({ role: "user", parts: [{ text: message }] });

  const response = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${process.env.GEMINI_API_KEY}`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        systemInstruction: {
          parts: [{
            text: `You are a friendly human-like WhatsApp conversation partner. Reply naturally, warmly and casually. Use simple everyday English. Keep replies short and conversational. Use emojis occasionally, not excessively. Understand the conversation before responding. Never sound like a customer service robot.`
          }]
        },
        contents: history.slice(-12),
        generationConfig: {
          temperature: 0.8,
          maxOutputTokens: 250
        }
      })
    }
  );

  if (!response.ok) {
    throw new Error("AI request failed: " + response.status);
  }

  const data = await response.json();
  const reply = data.candidates?.[0]?.content?.parts
    ?.map(part => part.text || "")
    .join("")
    .trim();

  if (!reply) throw new Error("Empty AI response");

  history.push({ role: "model", parts: [{ text: reply }] });
  conversations.set(jid, history.slice(-12));
  return reply;
}

async function startBot() {
  const { state, saveCreds } = await useMultiFileAuthState("./auth_info");

  const sock = makeWASocket({
    auth: state,
    browser: ["Ubuntu", "Chrome", "20.0.04"]
  });

  sock.ev.on("creds.update", saveCreds);

  // Backup: ask WhatsApp for a pairing code if PHONE_NUMBER is set
  if (!sock.authState.creds.registered && process.env.PHONE_NUMBER) {
    const phoneNumber = process.env.PHONE_NUMBER.replace(/[^0-9]/g, "");
    setTimeout(async () => {
      try {
        const code = await sock.requestPairingCode(phoneNumber);
        console.log("=================================");
        console.log("YOUR WHATSAPP PAIRING CODE: " + code);
        console.log("=================================");
      } catch (error) {
        console.error("Pairing code error:", error.message);
      }
    }, 5000);
  }

  sock.ev.on("connection.update", async (update) => {
    const { connection, lastDisconnect, qr } = update;

    if (qr) {
      latestQR = qr;
      console.log("New QR code ready. Open your Render address in a browser to scan it.");
    }

    if (connection === "open") {
      isOnline = true;
      latestQR = null;
      console.log("Bright AI WhatsApp bot is online!");
    }

    if (connection === "close") {
      isOnline = false;
      const code = lastDisconnect?.error?.output?.statusCode;
      if (code !== DisconnectReason.loggedOut) {
        console.log("Reconnecting...");
        startBot();
      } else {
        console.log("WhatsApp logged out. Please reconnect.");
      }
    }
  });

  sock.ev.on("messages.upsert", async ({ messages, type }) => {
    if (type !== "notify") return;

    for (const msg of messages) {
      if (!msg.message || msg.key.fromMe) continue;

      const jid = msg.key.remoteJid;
      if (!jid || jid === "status@broadcast") continue;

      const text =
        msg.message.conversation ||
        msg.message.extendedTextMessage?.text ||
        msg.message.imageMessage?.caption ||
        msg.message.videoMessage?.caption;

      if (!text?.trim()) continue;

      try {
        console.log("Incoming message:", text);
        const reply = await askAI(text, jid);
        await sock.sendMessage(jid, { text: reply });
        console.log("AI replied successfully.");
      } catch (error) {
        console.error("Reply error:", error.message);
      }
    }
  });
}

startBot().catch(console.error);

// Web server: shows the QR code in your browser so you can scan it
http
  .createServer(async (req, res) => {
    res.writeHead(200, { "Content-Type": "text/html; charset=utf-8" });

    if (isOnline) {
      res.end("<h2>Bot is online and linked to WhatsApp.</h2>");
    } else if (latestQR) {
      const image = await QRCode.toDataURL(latestQR, { width: 320 });
      res.end(
        `<meta http-equiv="refresh" content="20">` +
        `<h2>Scan this with WhatsApp: Settings, Linked Devices, Link a Device</h2>` +
        `<img src="${image}" alt="WhatsApp QR code">` +
        `<p>This page refreshes by itself every 20 seconds.</p>`
      );
    } else {
      res.end(
        `<meta http-equiv="refresh" content="5">` +
        `<h2>Starting up, the QR code will appear here in a few seconds...</h2>`
      );
    }
  })
  .listen(process.env.PORT || 3000);
