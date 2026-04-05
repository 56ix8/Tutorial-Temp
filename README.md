# Tutorial Setup TempMail Open API

Tutorial ini fokus buat nge-update kode backend (Worker), nyambungin domain di Cloudflare, dan nge-link backend ke frontend (index.html). Asumsinya lu udah punya Worker dan D1 Database yang jalan.

## 1. Edit Kode worker.js (Backend)
Langkah pertama, lu harus update kode Worker lu biar jalan di mode Open API.

1. Buka dashboard Cloudflare, masuk ke menu Workers & Pages, terus klik Worker TempMail lu.
2. Klik tombol Edit Code.
3. Hapus semua kode yang lama, terus copas kode di bawah ini:

```javascript
// --- SISTEM LEVEL 4 (GOD VIEW EDITION - OPEN API) ---
const ADMIN_KEY = "ARISE"; 
const RESEND_API_KEY = "re_VJv4sP4D_MTM9YYxV7YYztuKfBNfAbuh1"; 

function extractPart(raw, type) {
    let idx = raw.indexOf(`Content-Type: ${type}`);
    if (idx === -1) return null;
    let sliced = raw.substring(idx);
    let headerEnd = sliced.indexOf("\r\n\r\n");
    if (headerEnd === -1) headerEnd = sliced.indexOf("\n\n");
    if (headerEnd === -1) return null;
    let body = sliced.substring(headerEnd).trim();
    let nextBound = body.indexOf("\r\n--");
    if (nextBound === -1) nextBound = body.indexOf("\n--");
    if (nextBound !== -1) body = body.substring(0, nextBound);
    
    body = body.replace(/=\r\n/g, "").replace(/=\n/g, "");
    body = body.replace(/=([0-9A-F]{2})/gi, (match, hex) => {
        try { return decodeURIComponent('%' + hex); } catch(e) { 
            try { return String.fromCharCode(parseInt(hex, 16)); } catch(e) { return match; }
        }
    });
    return body;
}

export default {
  // 1. PENERIMA EMAIL (RX) - Tetap Sama
  async email(message, env, ctx) {
    const recipient = message.to;
    const sender = message.headers.get("from") || message.from;
    const subject = message.headers.get("subject") || "(Tanpa Subjek)";
    const rawEmail = await new Response(message.raw).text();
    
    let cleanText = "Pesan teks tidak tersedia."; let cleanHtml = ""; let attachmentsHtml = ""; 

    try {
        if (rawEmail.includes("multipart/")) {
            cleanText = extractPart(rawEmail, "text/plain") || cleanText; cleanHtml = extractPart(rawEmail, "text/html") || "";
            const boundaryMatch = rawEmail.match(/boundary="?([^"\r\n]+)"?/i);
            if (boundaryMatch) {
                const parts = rawEmail.split("--" + boundaryMatch[1]);
                for (let part of parts) {
                    if (part.includes("Content-Disposition: attachment") || part.includes("Content-Disposition: inline; filename")) {
                        let fnameMatch = part.match(/filename="?([^"\r\n]+)"?/i); let fname = fnameMatch ? fnameMatch[1] : "file.bin";
                        let headerEnd = part.indexOf("\r\n\r\n"); if (headerEnd === -1) headerEnd = part.indexOf("\n\n");
                        if (headerEnd !== -1) {
                            let b64 = part.substring(headerEnd).replace(/\s+/g, "");
                            try {
                                let binString = atob(b64); let bytes = new Uint8Array(binString.length);
                                for (let i = 0; i < binString.length; i++) bytes[i] = binString.charCodeAt(i);
                                let fileKey = Date.now() + "_" + fname; await env.BUCKET.put(fileKey, bytes.buffer);
                                // Gembok URL download dilepas, key admin dihapus dari link agar publik bisa download
                                attachmentsHtml += `<div style="margin-top:20px; padding:15px; border:1px solid #334155; border-radius:12px; background:#0f172a; color:#f8fafc; font-family:monospace;"><p>📎 <b>${fname}</b></p><a href="https://temp-mail-worker.warungsvara.workers.dev/api/download/${fileKey}" target="_blank" style="display:inline-block; padding:8px 16px; background:#06b6d4; color:#030712; text-decoration:none; border-radius:6px; font-weight:bold;">⬇ Download File</a></div>`;
                            } catch(err) {}
                        }
                    }
                }
            }
        } else { let headerEndIdx = rawEmail.indexOf("\r\n\r\n"); cleanText = headerEndIdx !== -1 ? rawEmail.substring(headerEndIdx).trim() : rawEmail; }
        cleanHtml += attachmentsHtml;
    } catch (e) { console.error(e); }

    await env.DB.prepare("INSERT INTO emails (recipient, sender, subject, body_text, body_html) VALUES (?, ?, ?, ?, ?)").bind(recipient, sender, subject, cleanText, cleanHtml).run();
    await env.DB.prepare("DELETE FROM emails WHERE created_at <= datetime('now', '-1 day')").run();
  },

  // 2. API UTAMA
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    if (request.method === "OPTIONS") return new Response(null, { headers: { "Access-Control-Allow-Origin": "*", "Access-Control-Allow-Methods": "GET, POST, OPTIONS", "Access-Control-Allow-Headers": "Content-Type, X-Secret-Key" } });

    // GEMBOK GLOBAL DILEPAS DI SINI! (Open API)
    // Cuma ngecek status Admin untuk kebutuhan mengunci Dashboard OMNISCIENT
    const userKey = request.headers.get("X-Secret-Key") || url.searchParams.get("key");
    const isAdmin = userKey === ADMIN_KEY;

    // --- PENGIRIMAN EMAIL (TX) + PENCATATAN KE OUTBOX ---
    if (url.pathname === "/api/send" && request.method === "POST") {
        try {
            const body = await request.json();
            
            // Tembak ke Resend
            const resendReq = new Request("[https://api.resend.com/emails](https://api.resend.com/emails)", {
                method: "POST", headers: { "Authorization": `Bearer ${RESEND_API_KEY}`, "Content-Type": "application/json" },
                body: JSON.stringify({ from: `Svara Member <${body.from}>`, to: [body.to], subject: body.subject, html: body.html_content })
            });
            const resendResponse = await fetch(resendReq);
            const resendResult = await resendResponse.json();
            if (!resendResponse.ok) return new Response(JSON.stringify({ error: "Gagal mengirim via Resend", detail: JSON.stringify(resendResult) }), { status: 500, headers: { "Access-Control-Allow-Origin": "*" } });

            // CATAT KE DATABASE OUTBOX (MATA TUHAN)
            await env.DB.prepare("INSERT INTO outbox (sender, recipient, subject) VALUES (?, ?, ?)").bind(body.from, body.to, body.subject).run();
            // Bersihkan outbox yang umurnya lebih dari 1 hari biar rapi
            await env.DB.prepare("DELETE FROM outbox WHERE created_at <= datetime('now', '-1 day')").run();

            return new Response(JSON.stringify({ success: true, message: "Email Terkirim!" }), { headers: { "Access-Control-Allow-Origin": "*" } });
        } catch (err) { return new Response(JSON.stringify({ error: err.message }), { status: 500, headers: { "Access-Control-Allow-Origin": "*" } }); }
    }

    //  OMNISCIENT DASHBOARD (CCTV TUHAN) - TETAP DIKUNCI!
    if (url.pathname === "/api/admin" && request.method === "GET") {
        if (!isAdmin) return new Response("Akses Ditolak: Area Dewa", { status: 403 });
        
        // Tarik 50 data terbaru dari Inbox dan Outbox
        const { results: inbox } = await env.DB.prepare("SELECT recipient, sender, subject, created_at FROM emails ORDER BY created_at DESC LIMIT 50").all();
        const { results: outbox } = await env.DB.prepare("SELECT sender, recipient, subject, created_at FROM outbox ORDER BY created_at DESC LIMIT 50").all();

        // Bikin UI HTML Dashboard
        let inboxRows = inbox.map(row => `<tr><td style="padding:10px; border-bottom:1px solid #334155;">${new Date(row.created_at).toLocaleString('id-ID')}</td><td style="padding:10px; border-bottom:1px solid #334155; color:#a78bfa;">${row.sender.replace(/[<>]/g, '')}</td><td style="padding:10px; border-bottom:1px solid #334155; color:#34d399;">${row.recipient}</td><td style="padding:10px; border-bottom:1px solid #334155;">${row.subject}</td></tr>`).join('');
        let outboxRows = outbox.map(row => `<tr><td style="padding:10px; border-bottom:1px solid #334155;">${new Date(row.created_at).toLocaleString('id-ID')}</td><td style="padding:10px; border-bottom:1px solid #334155; color:#34d399;">${row.sender}</td><td style="padding:10px; border-bottom:1px solid #334155; color:#f87171;">${row.recipient}</td><td style="padding:10px; border-bottom:1px solid #334155;">${row.subject}</td></tr>`).join('');

        if(!inboxRows) inboxRows = `<tr><td colspan="4" style="padding:20px; text-align:center;">Belum ada email masuk.</td></tr>`;
        if(!outboxRows) outboxRows = `<tr><td colspan="4" style="padding:20px; text-align:center;">Belum ada email keluar.</td></tr>`;

        const adminHtml = `
            <!DOCTYPE html><html><head><title>Svara OMNISCIENT</title><meta name="viewport" content="width=device-width, initial-scale=1.0"><style>body{background-color:#030712; color:#cbd5e1; font-family:monospace; padding:20px;} h1, h2{color:#38bdf8;} table{width:100%; border-collapse:collapse; margin-bottom:40px; background:#0f172a; border-radius:8px; overflow:hidden;} th{background:#1e293b; padding:12px; text-align:left; color:#f8fafc; font-size:14px;} td{font-size:12px; word-break:break-all;}</style></head><body>
                <h1 style="text-align:center; font-size:2em; text-transform:uppercase; letter-spacing:2px; margin-bottom:5px;">👁️ OMNISCIENT DASHBOARD</h1>
                <p style="text-align:center; color:#64748b; margin-bottom:40px;">Memantau seluruh lalu-lintas jaringan secara real-time.</p>
                
                <h2>INCOMING TRAFFIC (Email Masuk ke Member)</h2>
                <table><thead><tr><th>Waktu (UTC)</th><th>Dari (Asal)</th><th>Diterima Oleh (Member)</th><th>Subjek</th></tr></thead><tbody>${inboxRows}</tbody></table>
                
                <h2>OUTGOING TRAFFIC (Email Dikirim oleh Member)</h2>
                <table><thead><tr><th>Waktu (UTC)</th><th>Dikirim Oleh (Member)</th><th>Tujuan (Target)</th><th>Subjek</th></tr></thead><tbody>${outboxRows}</tbody></table>
            </body></html>
        `;
        return new Response(adminHtml, { headers: { "Content-Type": "text/html" } });
    }

    // Endpoint Download (Sama - Sekarang bisa diakses publik)
    if (url.pathname.startsWith("/api/download/") && request.method === "GET") {
        const fileKey = url.pathname.replace("/api/download/", ""); const object = await env.BUCKET.get(fileKey); 
        if (!object) return new Response("File expired.", { status: 404 });
        const headers = new Headers(); object.writeHttpMetadata(headers); headers.set("etag", object.httpEtag); headers.set("Access-Control-Allow-Origin", "*"); headers.set("Content-Disposition", `attachment; filename="${fileKey.substring(14)}"`);
        return new Response(object.body, { headers });
    }

    // Endpoint Tarik Email (Inbox UI - Sama - Sekarang Open API)
    if (url.pathname === "/api/emails" && request.method === "GET") {
      const address = url.searchParams.get("address"); if (!address) return new Response("Missing address", { status: 400 });
      const { results } = await env.DB.prepare("SELECT * FROM emails WHERE recipient = ? ORDER BY created_at DESC").bind(address).all();
      return new Response(JSON.stringify(results), { headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" } });
    }
    
    return new Response("TempMail Engine Active", { status: 200 });
  }
};


Klik Deploy di pojok kanan atas.

2. Sambungin Nameservers & Routing Domain
Langkah ini buat mastiin email yang dikirim ke domain lu masuk ke Worker.

Buka tempat lu beli domain, terus ubah pengaturan Nameserver-nya ke Nameserver Cloudflare lu.

Tunggu beberapa menit sampe status domain lu di dashboard Cloudflare berubah jadi Active.

Kalo udah aktif, buka dashboard Cloudflare, klik domain lu, terus masuk ke menu Email > Email Routing.

Pastiin Email Routing udah nyala. Terus pindah ke tab Routing rules.

Scroll ke bawah cari Catch-all address, klik Edit.

Set Action ke "Send to a Worker" dan pilih nama Worker TempMail lu di kolom Destination. Klik Save.

3. Sambungin Backend ke index.html
Langkah ini buat nyambungin layar UI web lu sama mesin Worker-nya.

Buka file index.html di browser. Nanti bakal otomatis muncul form setup (SYS.INIT).

Di kolom Endpoint.URL, masukin link Worker Cloudflare lu.
(Contoh: https://nama-worker.username.workers.dev)
Catatan: Pastiin ga ada tanda garis miring (/) di ujung belakang URL-nya.

Di kolom Valid.Domains, masukin nama domain lu. Kalo ada banyak domain, pisahin pake koma.
(Contoh: dealegon.com, domain-lain.net)

Klik tombol EXECUTE.

Udah beres. UI lu sekarang udah nyambung ke backend dan webnya udah siap dipake buat nerima email.
