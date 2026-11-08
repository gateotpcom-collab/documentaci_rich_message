# Dokumentasi Rich Message & Telegram Bot API 9.4

> Dikumpulkan dari hasil trial-and-error langsung ke server Telegram selama development
> bot Veldora Store (Agustus 2026). Fitur-fitur ini **sangat baru** dan belum ada
> dokumentasi resmi lengkap dari Telegram — semua di sini sudah diverifikasi jalan
> di server production.

---

## Bagian 1 — Bot API 9.4: Tombol Berwarna & Emoji Premium

### 1.1 Tombol Berwarna (`style`)

Setiap tombol inline (`InlineKeyboardButton`) sekarang punya field `style` opsional:

```js
{
  text: "Beli Sekarang",
  callback_data: "buy_1",
  style: "success"   // "primary" | "success" | "danger"
}
```

| Style     | Warna (kira-kira) | Dipakai buat                          |
|-----------|--------------------|-----------------------------------------|
| `primary` | Biru               | Navigasi, pilihan netral, "Lanjut"      |
| `success` | Hijau              | Aksi positif — beli, konfirmasi, tambah |
| `danger`  | Merah              | Aksi berbahaya/batal — hapus, kembali, batal |

Kalau `style` tidak diisi, tombol tampil normal (abu-abu/default), tidak ada style aktif.

`style` juga bisa dipasang di tombol URL, bukan cuma `callback_data` — dua-duanya boleh punya `style`.

### 1.2 Emoji Premium di Tombol (`icon_custom_emoji_id`)

```js
{
  text: "VPS Digital Ocean",           // TANPA emoji di teks
  callback_data: "buyvps_start",
  icon_custom_emoji_id: "5361897938209947023"  // ID custom emoji premium
}
```

- Emoji custom **ditaruh sebagai ikon terpisah**, bukan di dalam teks tombol.
- ID custom emoji itu ID sticker/emoji premium ASLI dari Telegram — **bukan angka bebas**.
  Kalau ID salah/ngarang, Telegram balikin error `CUSTOM_EMOJI_ID_INVALID` dan tombolnya
  gagal total (bukan sekadar fallback ke emoji biasa).
- **Cara dapetin ID emoji premium**: forward pesan berisi emoji itu (dari akun Telegram
  Premium) ke bot semacam `@RawDataBot` / `@ShowJsonBot`, lalu cari field
  `custom_emoji_id` di JSON hasil forward.
- Restriction penting: bot **hanya diizinkan** pakai custom emoji (baik di teks maupun
  ikon tombol) kalau **owner bot** (akun yang daftar bot ke BotFather) punya Telegram
  Premium, DAN cuma berlaku di **private chat, group, supergroup** — **CHANNEL TIDAK
  TERMASUK**. Custom emoji yang dikirim bot langsung ke channel akan gagal/di-strip.

### 1.3 Emoji Premium di Teks Biasa (`<tg-emoji>`)

Untuk pesan/caption biasa (bukan rich message), custom emoji dipasang lewat tag HTML:

```html
<tg-emoji emoji-id="5361897938209947023">🖥️</tg-emoji>
```

- Wajib `parse_mode: "HTML"`.
- Isi tag (`🖥️`) adalah fallback emoji biasa buat klien yang gak support custom emoji.
- Sama kayak di atas: butuh ID asli, dan kena batasan private/group/supergroup only
  (gak bisa ke channel).

---

## Bagian 2 — Rich Message (Bot API 10.1 & 10.2)

### 2.1 Method: `sendRichMessage`

Endpoint baru, terpisah dari `sendMessage`. Dipanggil via raw HTTPS POST (library lama
kayak Telegraf belum punya wrapper buat ini):

```
POST https://api.telegram.org/bot<TOKEN>/sendRichMessage
Content-Type: application/json

{
  "chat_id": "123456789",
  "rich_message": { "html": "<h1>Judul</h1><p>Isi pesan</p>" },
  "reply_markup": { "inline_keyboard": [[ { "text": "Tombol", "callback_data": "x" } ]] }
}
```

**Field kunci**: `rich_message` adalah objek dengan **PERSIS SATU** dari dua field ini:
- `html` — string HTML (lihat tag yang didukung di bawah)
- `markdown` — string Markdown (alternatif, belum dieksplorasi mendalam di dok ini)

> ⚠️ **JANGAN** pakai struktur `blocks: [...]` (array objek JSON dengan `type`
> discriminator). Itu ternyata BUKAN cara yang benar — sempat dicoba dan selalu
> gagal dengan berbagai error (`Can't find field "type"`, `RICH_MESSAGE_CONTENT_REQUIRED`,
> dll). Cara yang benar dan terbukti jalan adalah string HTML seperti di atas.

### 2.2 Method: Edit Rich Message (lewat `editMessageText`)

Ini yang paling gampang kelewat: **tidak ada** method `editRichMessage` terpisah.
`editMessageText` (method lama) dapat parameter baru `rich_message`:

```
POST https://api.telegram.org/bot<TOKEN>/editMessageText
Content-Type: application/json

{
  "chat_id": "123456789",
  "message_id": 42,
  "rich_message": { "html": "<h1>Judul Baru</h1>" },
  "reply_markup": { ... }
}
```

- Field `text` dan `rich_message` **saling eksklusif** — isi salah satu saja.
- Ini jauh lebih murah/cepat dibanding hapus pesan lama + kirim baru (yang sempat jadi
  workaround sebelum nemu ini). Selalu coba **edit dulu**, baru fallback ke
  delete+kirim-baru kalau edit gagal (biasanya karena pesan sebelumnya bukan rich
  message, jadi "gak bisa diubah tipenya").

### 2.3 Tag HTML yang Didukung di `rich_message.html`

Berdasarkan spec resmi + hasil trial:

| Tag | Fungsi |
|---|---|
| `<h1>` ... `<h6>` | Heading, 1 = paling besar |
| `<p>`, `<i>`, `<b>`, `<u>`, `<s>`, `<code>` | Paragraf & format teks standar (sama kayak parse_mode HTML biasa) |
| `<blockquote>` | Kutipan blok |
| `<table bordered striped>` | Tabel. Attribute `bordered`/`striped` boolean (tulis nama attribute-nya aja, gak perlu `="true"`) |
| `<tr>`, `<th>`, `<td>` | Baris/header/sel tabel — cuma boleh isi **inline formatting**, gak boleh nested block element (tabel dalam tabel, dst) |
| `<details>` / `<details open>` | Konten collapsible. Tanpa `open` = collapsed default (bagus buat "ringkas dulu, tap baru detail") |
| `<summary>` | Judul yang selalu keliatan di `<details>`, di-tap buat expand |
| `<img src="URL" />` | Foto tunggal. **Wajib HTTP/HTTPS URL**, gak bisa upload file langsung (karena payload-nya JSON, bukan multipart) |
| `<video src="URL"></video>` | Video. Sama, wajib URL, JANGAN pakai attribute `poster` (gak ada di spec, bikin gagal) |
| `<audio src="URL"></audio>` | Audio |
| `<figure>...</figure>` | Bungkus media + caption |
| `<figcaption>` | Caption di dalam `<figure>` |
| `<tg-slideshow><img/><img/>...</tg-slideshow>` | Galeri foto yang bisa di-swipe. Isinya beberapa `<img>` berurutan |
| `<ul>`, `<ol>`, `<li>` | List biasa & bernomor |
| `<tg-emoji emoji-id="...">` | Custom emoji premium (sama kayak di teks biasa) |

**Batasan resmi** (dari spec):
- Maks 500 block (termasuk nested)
- Maks 20 kolom tabel
- Maks 50 lampiran media
- Maks 16 level nesting
- Maks 32.768 karakter UTF-8

### 2.4 Media butuh nested object dengan `type`

Kalau bikin block via cara lain (bukan HTML string), field media (foto/video) bukan
string URL polos — harus dibungkus:

```json
{ "type": "photo", "photo": { "type": "photo", "media": "https://..." } }
```

(Ini best-practice buat opsi `blocks` JSON kalau suatu saat perlu, tapi sekali lagi:
untuk kebanyakan kasus, **string HTML jauh lebih simpel dan terbukti reliable**.)

### 2.5 Batasan Channel

Sama kayak custom emoji, `sendRichMessage` dengan konten yang butuh custom emoji/media
kaya **kemungkinan besar gak bisa dikirim LANGSUNG ke channel**. Workaround yang
kepake di project ini: kirim dulu ke **private chat** (misal ke owner), baru
**forward** pesan itu ke channel — forward gak kena batasan yang sama kayak "bot
kirim langsung".

### 2.6 Bonus: `sendRichMessageDraft`

Ada method buat streaming rich message secara bertahap (efek kayak "sedang mengetik"),
cuma jalan di private chat, hasil draft-nya ephemeral (~30 detik) sebelum harus
di-finalize pakai `sendRichMessage` beneran. Belum diimplementasi di project ini,
tapi berpotensi bagus buat animasi/loading state yang lebih hidup.

---

## Bagian 3 — Pola Praktis yang Dipakai di Project Ini

### Helper yang dipakai (lihat `index.js`)
- `sendRichMessage(chatId, richMessage, replyMarkup)` — kirim baru
- `editRichMessage(chatId, messageId, richMessage, replyMarkup)` — edit di tempat
- `sendRichMenu(ctx, buildHtmlFn, keyboard, fallbackFn, label)` — helper level-tinggi:
  coba edit dulu → kalau gagal, hapus+kirim baru → kalau masih gagal, jalanin
  `fallbackFn` (versi non-rich lama) biar user gak pernah lihat error mentah.

### Kenapa selalu ada fallback?
Fitur ini masih sangat baru (dirilis Juni–Juli 2026) dan minim dokumentasi publik.
Prinsip yang dipakai: **setiap pemanggilan rich message harus punya jaring pengaman**
ke tampilan lama (teks/foto biasa) supaya kalaupun Telegram berubah perilaku atau ada
skema yang belum ketemu, user tetap dapat tampilan yang jalan — bukan pesan kosong
atau error.

---

*Dokumen ini akan terus di-update seiring ditemukannya detail baru dari trial-and-error
lanjutan. Terakhir diperbarui: Agustus 2026.*
