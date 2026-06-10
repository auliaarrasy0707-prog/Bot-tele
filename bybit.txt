/**
 * Monzo Report Bot (Termux Edition) - Per Target Terpisah (Fixed Loop)
 * - /start : info akun + tombol Report
 * - Owner  : Add Sender Gmail, Add Access, List, dll.
 * - Report : Mengirim email terpisah SATU PER SATU ke setiap target (1 email = 1 target)
 *
 * Jalankan: node testing
 */

const fs = require("fs");
const path = require("path");
const TelegramBot = require("node-telegram-bot-api");
const nodemailer = require("nodemailer");

// ---- Load .env sederhana ----
(function loadDotEnv() {
  const p = path.join(__dirname, ".env");
  if (!fs.existsSync(p)) return;
  for (const line of fs.readFileSync(p, "utf8").split("\n")) {
    const m = line.match(/^\s*([A-Z0-9_]+)\s*=\s*(.*)\s*$/i);
    if (m && !process.env[m[1]]) process.env[m[1]] = m[2].replace(/^["']|["']$/g, "");
  }
})();

// ---- Konfigurasi Token & Owner ----
const BOT_TOKEN = "8616785901:AAEgqbX2P2DQNAzqf8PO-SBD2yWVGqwdYw8";
const OWNER_ID = 7482623335;

// ---- Daftar Target Email (TERBARU) ----
const TARGET_EMAILS = [
  "support@bybit.com",
  "compliance@bybit.com",
  "support@telegram.org",
  "dmca@telegram.org",
  "security@telegram.org",
  "report-updates@netcraft.com",
  "scam@netcraft.com"
];

// 🟢 TAMBAHKAN BARIS INI DI BAWAH TARGET_EMAILS:
const TOTAL_LOOP = 20; // Ganti angka 15 ini sesukamu (misal: 10, 20, atau 50)


if (!BOT_TOKEN) {
  console.error("❌ BOT_TOKEN wajib diisi di dalam script!");
  process.exit(1);
}

// ---- Penyimpanan data.json ----
const DATA_FILE = path.join(__dirname, "data.json");
const defaultData = { senders: [], access: [] };
let data = defaultData;
try {
  if (fs.existsSync(DATA_FILE)) data = { ...defaultData, ...JSON.parse(fs.readFileSync(DATA_FILE, "utf8")) };
} catch (e) {
  console.error("⚠️ data.json rusak, reset.", e.message);
  data = defaultData;
}
function saveData() {
  fs.writeFileSync(DATA_FILE, JSON.stringify(data, null, 2));
}

// ---- Helper akses ----
const isOwner = (id) => Number(id) === OWNER_ID;
const hasAccess = (id) => isOwner(id) || data.access.some((u) => Number(u.id) === Number(id));

// ---- Bot ----
const bot = new TelegramBot(BOT_TOKEN, { polling: true });

const sessions = new Map();
function setSession(uid, s) { sessions.set(uid, s); }
function getSession(uid) { return sessions.get(uid); }
function clearSession(uid) { sessions.delete(uid); }

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

// ---- Tampilan ----
function startKeyboard(uid) {
  const rows = [[{ text: "📝 Report", callback_data: "report" }]];
  if (isOwner(uid)) rows.push([{ text: "👑 Owner Menu", callback_data: "owner" }]);
  return { reply_markup: { inline_keyboard: rows } };
}

function ownerKeyboard() {
  return {
    reply_markup: {
      inline_keyboard: [
        [{ text: "➕ Add Sender Gmail", callback_data: "add_sender" }],
        [{ text: "📋 List Sender", callback_data: "list_sender" }],
        [{ text: "➕ Add Access", callback_data: "add_access" }],
        [{ text: "📋 List Access", callback_data: "list_access" }],
        [{ text: "⬅️ Back", callback_data: "back_start" }],
      ],
    },
  };
}

function accountInfoText(user) {
  const role = isOwner(user.id) ? "Owner 👑" : hasAccess(user.id) ? "User ✅" : "Guest ❌";
  const fullName = ((user.first_name || "") + " " + (user.last_name || "")).trim();
  return (
    `مرحبًا بوت تقرير حساب مزيف أو احتيا\n\n` +
    `*Bot Report Fake Account Or This Label Aims To Avoid The Hassle Of Manual Reporting This Bot Was Created By ABDY*\n\n` +
    `🤖 *Bot Report Account Fake Or Label*\n\n` +
    `👤 *Account Info*\n` +
    `• Name : ${fullName || "𝗮𝗯𝗱𝘆👋"}\n` +
    `• User : @${user.username || "Morkelzabdy"}\n` +
    `• ID   : ${user.id}\n` +
    `• Role : ${role}\n\n` +
    `🎯 *Targets* : target who just knows\n` +
    `📬 *Total Sender* : 132 Sender On`
  );
}

// ---- Perintah Dasar ----
bot.onText(/^\/start$/, (msg) => {
  clearSession(msg.from.id);
  bot.sendMessage(msg.chat.id, accountInfoText(msg.from), {
    parse_mode: "Markdown",
    ...startKeyboard(msg.from.id),
  });
});

bot.onText(/^\/owner$/, (msg) => {
  if (!isOwner(msg.from.id)) return bot.sendMessage(msg.chat.id, "❌ Menu khusus owner.");
  bot.sendMessage(msg.chat.id, "👑 *Owner Menu*", { parse_mode: "Markdown", ...ownerKeyboard() });
});

bot.onText(/^\/cancel$/, (msg) => {
  clearSession(msg.from.id);
  bot.sendMessage(msg.chat.id, "✅ Dibatalkan.", startKeyboard(msg.from.id));
});

// ---- Callback Query ----
bot.on("callback_query", async (q) => {
  const uid = q.from.id;
  const chatId = q.message.chat.id;
  const dataCb = q.data;

  try { await bot.answerCallbackQuery(q.id); } catch {}

  if (dataCb === "back_start") {
    return bot.sendMessage(chatId, accountInfoText(q.from), { parse_mode: "Markdown", ...startKeyboard(uid) });
  }

  if (dataCb === "owner") {
    if (!isOwner(uid)) return bot.sendMessage(chatId, "❌ Khusus owner.");
    return bot.sendMessage(chatId, "👑 *Owner Menu*", { parse_mode: "Markdown", ...ownerKeyboard() });
  }

  if (dataCb === "report") {
    if (!hasAccess(uid)) return bot.sendMessage(chatId, "❌ Kamu belum punya akses. Hubungi owner.");
    if (data.senders.length === 0)
      return bot.sendMessage(chatId, "⚠️ Belum ada sender Gmail. Owner harus Add Sender dulu.");
    setSession(uid, { step: "subject", payload: { photos: [] } });
    return bot.sendMessage(chatId, "📝 Kirim *Subject* email:\n(Ketik /cancel untuk batal)", { parse_mode: "Markdown" });
  }

  if (dataCb.startsWith("pick_sender:")) {
    const s = getSession(uid);
    if (!s || s.step !== "pick_sender") return;
    const idx = Number(dataCb.split(":")[1]);
    s.payload.selected = s.payload.selected || new Set();
    if (s.payload.selected.has(idx)) s.payload.selected.delete(idx);
    else s.payload.selected.add(idx);
    return bot.editMessageReplyMarkup(senderPickKeyboard(s.payload.selected).reply_markup, {
      chat_id: chatId,
      message_id: q.message.message_id,
    });
  }

  if (dataCb === "send_now") {
    const s = getSession(uid);
    if (!s || s.step !== "pick_sender") return;
    const selected = Array.from(s.payload.selected || []);
    if (selected.length === 0) return bot.sendMessage(chatId, "❌ Pilih minimal 1 sender.");
    
    const infoMsg = await bot.sendMessage(chatId, `📤 Memulai blast terpisah ke ${TARGET_EMAILS.length} target secara bergantian...`);
    const results = [];
    
    for (const idx of selected) {
      const sender = data.senders[idx];
      if (!sender) continue;
      
      const totalSent = await sendEmailOneByOne(sender, s.payload);
      sender.used = (sender.used || 0) + totalSent;
      saveData();
      results.push(`🔹 *${sender.email}* berhasil mengirim ke *${totalSent} / ${TARGET_EMAILS.length}* target.`);
    }
    
    clearSession(uid);
    try { await bot.deleteMessage(chatId, infoMsg.message_id); } catch {}
    
    return bot.sendMessage(chatId, "📬 *Hasil Akhir Pengiriman:*\n\nSelesai memproses antrean target secara terpisah.", {
      parse_mode: "Markdown",
      ...startKeyboard(uid),
    });
  }

  if (dataCb === "cancel_session") {
    clearSession(uid);
    return bot.sendMessage(chatId, "✅ Dibatalkan.", startKeyboard(uid));
  }

  // ---- Fitur Owner ----
  if (!isOwner(uid)) return;

  if (dataCb === "add_sender") {
    setSession(uid, { step: "add_sender_email", payload: {} });
    return bot.sendMessage(chatId, "📧 Kirim alamat *Gmail* yang mau ditambahkan:\n(/cancel untuk batal)", { parse_mode: "Markdown" });
  }

  if (dataCb === "list_sender") {
    if (data.senders.length === 0) return bot.sendMessage(chatId, "Kosong.");
    const rows = data.senders.map((s, i) => [{
      text: `🟢 ${s.email} (Total Kirim: ${s.used || 0})`,
      callback_data: `del_sender:${i}`,
    }]);
    rows.push([{ text: "⬅️ Back", callback_data: "owner" }]);
    return bot.sendMessage(chatId, "📋 *List Sender* (klik untuk hapus):", {
      parse_mode: "Markdown",
      reply_markup: { inline_keyboard: rows },
    });
  }

  if (dataCb.startsWith("del_sender:")) {
    const i = Number(dataCb.split(":")[1]);
    const removed = data.senders.splice(i, 1)[0];
    saveData();
    return bot.sendMessage(chatId, `🗑️ Dihapus: ${removed?.email || "?"}`);
  }

  if (dataCb === "add_access") {
    setSession(uid, { step: "add_access_id", payload: {} });
    return bot.sendMessage(chatId, "🆔 Kirim *Telegram user ID*:\n(/cancel untuk batal)", { parse_mode: "Markdown" });
  }

  if (dataCb === "list_access") {
    if (data.access.length === 0) return bot.sendMessage(chatId, "Kosong.");
    const rows = data.access.map((u, i) => [{
      text: `❌ ${u.id} ${u.note ? "- " + u.note : ""}`,
      callback_data: `del_access:${i}`,
    }]);
    rows.push([{ text: "⬅️ Back", callback_data: "owner" }]);
    return bot.sendMessage(chatId, "📋 *List Access* (klik untuk hapus):", {
      parse_mode: "Markdown",
      reply_markup: { inline_keyboard: rows },
    });
  }

  if (dataCb.startsWith("del_access:")) {
    const i = Number(dataCb.split(":")[1]);
    const removed = data.access.splice(i, 1)[0];
    saveData();
    return bot.sendMessage(chatId, `🗑️ Akses dihapus: ${removed?.id}`);
  }
});

// ---- Penanganan Langkah Input ----
bot.on("message", async (msg) => {
  if (!msg.from || msg.text?.startsWith("/")) return;
  const uid = msg.from.id;
  const chatId = msg.chat.id;
  const s = getSession(uid);
  if (!s) return;

  if (s.step === "add_sender_email") {
    const email = (msg.text || "").trim().toLowerCase();
    if (!/^[^\s@]+@gmail\.com$/.test(email)) return bot.sendMessage(chatId, "❌ Harus alamat @gmail.com. Coba lagi atau /cancel.");
    s.payload.email = email;
    s.step = "add_sender_pass";
    return bot.sendMessage(chatId, "🔑 Sekarang kirim *App Password* Gmail (16 karakter):", { parse_mode: "Markdown" });
  }

  if (s.step === "add_sender_pass") {
    const pass = (msg.text || "").trim().replace(/\s+/g, "");
    if (pass.length < 12) return bot.sendMessage(chatId, "❌ Password terlalu pendek. Coba lagi atau /cancel.");
    try {
      const t = nodemailer.createTransport({ service: "gmail", auth: { user: s.payload.email, pass } });
      await t.verify();
    } catch (e) {
      clearSession(uid);
      return bot.sendMessage(chatId, "❌ Gagal login Gmail: " + e.message);
    }
    data.senders.push({ email: s.payload.email, pass, used: 0, addedAt: Date.now() });
    saveData();
    clearSession(uid);
    return bot.sendMessage(chatId, `✅ Sender ditambahkan: ${s.payload.email}`, ownerKeyboard());
  }

  if (s.step === "add_access_id") {
    const id = Number((msg.text || "").trim());
    if (!id) return bot.sendMessage(chatId, "❌ ID harus angka. Coba lagi atau /cancel.");
    s.payload.id = id;
    s.step = "add_access_note";
    return bot.sendMessage(chatId, "📝 Kirim catatan nama (atau ketik - untuk skip):", { parse_mode: "Markdown" });
  }

  if (s.step === "add_access_note") {
    const note = (msg.text || "").trim();
    data.access.push({ id: s.payload.id, note: note === "-" ? "" : note, addedAt: Date.now() });
    saveData();
    clearSession(uid);
    return bot.sendMessage(chatId, `✅ Akses diberikan ke ${s.payload.id}`, ownerKeyboard());
  }

  if (s.step === "subject") {
    if (!msg.text) return bot.sendMessage(chatId, "❌ Kirim teks subject.");
    s.payload.subject = msg.text.trim();
    s.step = "body";
    return bot.sendMessage(chatId, "✍️ Sekarang kirim *isi laporan*:", { parse_mode: "Markdown" });
  }

  if (s.step === "body") {
    if (!msg.text) return bot.sendMessage(chatId, "❌ Kirim teks isi laporan.");
    s.payload.body = msg.text.trim();
    s.step = "photos";
    return bot.sendMessage(chatId, "📷 Kirim *foto support*.\nKetik `done` kalau sudah, atau `skip` jika tanpa foto.", { parse_mode: "Markdown" });
  }

  if (s.step === "photos") {
    if (msg.photo && msg.photo.length) {
      const fileId = msg.photo[msg.photo.length - 1].file_id;
      try {
        const link = await bot.getFileLink(fileId);
        const res = await fetch(link);
        const buf = Buffer.from(await res.arrayBuffer());
        s.payload.photos.push({ filename: `photo_${s.payload.photos.length + 1}.jpg`, content: buf });
        return bot.sendMessage(chatId, `✅ Foto #${s.payload.photos.length} diterima. Ketik \`done\` jika sudah selesai.`, { parse_mode: "Markdown" });
      } catch (e) {
        return bot.sendMessage(chatId, "❌ Gagal mengambil foto: " + e.message);
      }
    }
    if (msg.text && /^(done|skip)$/i.test(msg.text.trim())) {
      s.step = "pick_sender";
      s.payload.selected = new Set();
      return bot.sendMessage(
        chatId,
        `📧 Pilih sender. Tekan *Send* untuk mulai blast satu per satu.\nLampiran: ${s.payload.photos.length} foto`,
        { parse_mode: "Markdown", ...senderPickKeyboard(s.payload.selected) }
      );
    }
    return bot.sendMessage(chatId, "Kirim foto, atau ketik done / skip.");
  }
});

function senderPickKeyboard(selectedSet) {
  const rows = data.senders.map((s, i) => {
    const mark = selectedSet.has(i) ? "✅" : "⬜";
    return [{ text: `${mark} ${s.email}`, callback_data: `pick_sender:${i}` }];
  });
  rows.push([
    { text: "📤 Send", callback_data: "send_now" },
    { text: "❌ Cancel", callback_data: "cancel_session" },
  ]);
  return { reply_markup: { inline_keyboard: rows } };
}

async function sendEmailOneByOne(sender, payload) {
  let successCount = 0;

  for (const target of TARGET_EMAILS) {
    const transporter = nodemailer.createTransport({
      service: "gmail",
      auth: { user: sender.email, pass: sender.pass },
    });

    try {
      await transporter.sendMail({
        from: sender.email,
        to: target, 
        subject: payload.subject,
        text: payload.body,
        attachments: payload.photos.map((p) => ({ filename: p.filename, content: p.content })),
      });
      successCount++;
      console.log(`[SUKSES] Terkirim dari ${sender.email} ke target: ${target}`);
    } catch (err) {
      console.error(`[GAGAL] Mengirim ke ${target}:`, err.message);
    }

    await sleep(1200);
  }

  return successCount;
}

bot.on("polling_error", (e) => console.error("polling_error:", e.message));
process.on("unhandledRejection", (e) => console.error("unhandledRejection:", e));

// ---- TAMPILAN STARTUP TERMINAL ----
console.log(" ___________________________________________________________");
console.log("|                                                           |");
console.log("|    👑 SAYA ADALAH BOT REPORT DIDIKAN KING ABDY            |");
console.log("|    ABDY ADALAH ORANG GANTENG SE-DUNIA 👑                 |");
console.log("|___________________________________________________________|");
console.log(`🤖 Bot aktif. Menggunakan mode antrean ${TARGET_EMAILS.length} target terpisah siap.`);
