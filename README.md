# Tutorial Setup TempMail Open API

Tutorial ini fokus buat nge-update kode backend (Worker), nyambungin domain di Cloudflare, dan nge-link backend ke frontend (index.html). Asumsinya lu udah punya Worker dan D1 Database yang jalan.

## 1. Edit Kode worker.js (Backend)
Langkah pertama, lu harus update kode Worker lu biar jalan di mode Open API.

1. Buka dashboard Cloudflare, masuk ke menu Workers & Pages, terus klik Worker TempMail lu.
2. Klik tombol Edit Code.
3. Hapus semua kode yang lama, terus copas kode di bawah ini:

```javascript
// --- SISTEM TEMPMAIL (SUPER CLEAN - OPEN API) ---

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
  // 1. PENERIMA EMAIL (RX)
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
                                let fileKey = Date.now() + "_" + fname; 
                                
                                // Simpan ke R2 kalau ada bindingnya
                                if (env.BUCKET) {
                                    await env.BUCKET.put(fileKey, bytes.buffer);
                                    attachmentsHtml += `<div style="margin-top:20px; padding:15px; border:1px solid #334155; border-radius:12px; background:#0f172a; color:#f8fafc; font-family:monospace;"><p>📎 <b>${fname}</b></p><a href="/api/download/${fileKey}" target="_blank" style="display:inline-block; padding:8px 16px; background:#06b6d4; color:#030712; text-decoration:none; border-radius:6px; font-weight:bold;">⬇ Download File</a></div>`;
                                }
                            } catch(err) {}
                        }
                    }
                }
            }
        } else { let headerEndIdx = rawEmail.indexOf("\r\n\r\n"); cleanText = headerEndIdx !== -1 ? rawEmail.substring(headerEndIdx).trim() : rawEmail; }
        cleanHtml += attachmentsHtml;
    } catch (e) { console.error(e); }

    // Simpan ke DB Inbox & Hapus yang udah lewat 1 hari
    await env.DB.prepare("INSERT INTO emails (recipient, sender, subject, body_text, body_html) VALUES (?, ?, ?, ?, ?)").bind(recipient, sender, subject, cleanText, cleanHtml).run();
    await env.DB.prepare("DELETE FROM emails WHERE created_at <= datetime('now', '-1 day')").run();
  },

  // 2. API UTAMA (BACA EMAIL & DOWNLOAD)
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // Handle CORS biar web bisa narik data
    if (request.method === "OPTIONS") {
        return new Response(null, { headers: { "Access-Control-Allow-Origin": "*", "Access-Control-Allow-Methods": "GET, POST, OPTIONS", "Access-Control-Allow-Headers": "Content-Type" } });
    }

    // Endpoint Download Lampiran
    if (url.pathname.startsWith("/api/download/") && request.method === "GET") {
        if (!env.BUCKET) return new Response("Storage tidak dikonfigurasi.", { status: 500 });
        const fileKey = url.pathname.replace("/api/download/", ""); 
        const object = await env.BUCKET.get(fileKey); 
        if (!object) return new Response("File expired atau tidak ditemukan.", { status: 404 });
        
        const headers = new Headers(); 
        object.writeHttpMetadata(headers); 
        headers.set("etag", object.httpEtag); 
        headers.set("Access-Control-Allow-Origin", "*"); 
        headers.set("Content-Disposition", `attachment; filename="${fileKey.substring(14)}"`);
        return new Response(object.body, { headers });
    }

    // Endpoint Tarik Email (Inbox UI - Mode Terbuka / Open API)
    if (url.pathname === "/api/emails" && request.method === "GET") {
      const address = url.searchParams.get("address"); 
      if (!address) return new Response("Missing address", { status: 400 });
      
      const { results } = await env.DB.prepare("SELECT * FROM emails WHERE recipient = ? ORDER BY created_at DESC").bind(address).all();
      return new Response(JSON.stringify(results), { headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" } });
    }
    
    return new Response("TempMail Engine Active", { status: 200, headers: { "Access-Control-Allow-Origin": "*" } });
  }
};
````

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
