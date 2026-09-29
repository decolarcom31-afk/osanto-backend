// ============================================================
// OSANTO DISPARADOR EMAIL — Backend de envio via SendPulse
// Node.js 18+ (fetch nativo). Sem dependências externas.
// Deploy: Render (Web Service) — Start: node server.js
// ============================================================

const http = require("http");

// ===== CONFIGURAÇÃO =====
const SENDPULSE_ID = process.env.SENDPULSE_ID || "8208c9046241a0a7673a612abf36df54";
const SENDPULSE_SECRET = process.env.SENDPULSE_SECRET || "693594afebc65ba117e3e15dd2f54820";

// ⚠️ Remetentes: troque pelo e-mail JÁ VERIFICADO na SendPulse
// (Configurações → SMTP → Senders). E-mail Gmail costuma ser recusado.
const SENDERS = [
  { from_email: "contato@seudominio.com.br", from_name: "OSANTO" }
];

const PORT = process.env.PORT || 3000;
const INTERVALO_MS = 1200; // pausa entre envios
// =========================

let cachedToken = null;
let tokenExpiresAt = 0;

// --- OAuth2: obtém o token de acesso ---
async function obterToken() {
  if (cachedToken && Date.now() < tokenExpiresAt) return cachedToken;
  const res = await fetch("https://api.sendpulse.com/oauth/access_token", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      grant_type: "client_credentials",
      client_id: SENDPULSE_ID,
      client_secret: SENDPULSE_SECRET,
    }),
  });
  const data = await res.json().catch(() => ({}));
  if (!data.access_token) {
    throw new Error(data.error_description || data.error || "Falha ao obter token");
  }
  cachedToken = data.access_token;
  tokenExpiresAt = Date.now() + ((data.expires_in || 3560) - 60) * 1000;
  return cachedToken;
}

// --- Envia 1 e-mail via SMTP da SendPulse ---
async function enviarEmail({ destinatario, assunto, corpo }) {
  const token = await obterToken();
  const remetente = SENDERS[Math.floor(Math.random() * SENDERS.length)];

  const res = await fetch("https://api.sendpulse.com/smtp/emails", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: "Bearer " + token,
    },
    body: JSON.stringify({
      email: {
        html: corpo,
        subject: assunto,
        from: { name: remetente.from_name, email: remetente.from_email },
        to: [{ email: destinatario }],
      },
    }),
  });

  const data = await res.json().catch(() => ({}));
  if (data.error) {
    const msg = data.error.error_description || data.error_description || data.error;
    throw new Error(msg);
  }
  return { ok: true, remetente: remetente.from_email };
}

// --- Servidor HTTP ---
const server = http.createServer(async (req, res) => {
  res.setHeader("Access-Control-Allow-Origin", "*");
  res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS");
  res.setHeader("Access-Control-Allow-Headers", "Content-Type");

  if (req.method === "OPTIONS") { res.writeHead(204); return res.end(); }

  // Health check (endpoint raiz)
  if (req.method === "GET" && (req.url === "/" || req.url === "/health")) {
    res.writeHead(200, { "Content-Type": "application/json" });
    return res.end(JSON.stringify({ ok: true, status: "OSANTO BACKEND ONLINE" }));
  }

  // Envio
  if (req.method === "POST" && req.url === "/send") {
    let body = "";
    req.on("data", (c) => (body += c));
    req.on("end", async () => {
      try {
        const { destinatario, quantidade, assunto, corpo } = JSON.parse(body);
        const qtd = Number(quantidade);
        if (!destinatario || !Number.isInteger(qtd) || qtd <= 0) {
          res.writeHead(400, { "Content-Type": "application/json" });
          return res.end(JSON.stringify({ ok: false, erro: "Dados inválidos" }));
        }

        const resultados = [];
        for (let i = 1; i <= qtd; i++) {
          try {
            const r = await enviarEmail({ destinatario, assunto, corpo });
            resultados.push({ n: i, ok: true, remetente: r.remetente });
          } catch (e) {
            resultados.push({ n: i, ok: false, erro: e.message });
          }
          if (i < qtd) await new Promise((r) => setTimeout(r, INTERVALO_MS));
        }

        res.writeHead(200, { "Content-Type": "application/json" });
        res.end(JSON.stringify({ ok: true, resultados }));
      } catch (e) {
        res.writeHead(500, { "Content-Type": "application/json" });
        res.end(JSON.stringify({ ok: false, erro: e.message }));
      }
    });
    return;
  }

  res.writeHead(404, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ ok: false, erro: "Rota não encontrada" }));
});

server.listen(PORT, () => {
  console.log("✅ OSANTO backend rodando na porta " + PORT);
});
