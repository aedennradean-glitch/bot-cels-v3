# Cels V3 — Railway Setup

## 1. Persiapan

- Gunakan Node.js 20.x (sesuai `package.json`).
- Upload `index.js` dan `package.json` ke repository GitHub yang terhubung ke Railway.
- Pastikan bot memiliki izin melihat/mengirim pesan pada channel teks dan izin Connect pada voice channel.
- Aktifkan **Message Content Intent** di Discord Developer Portal.
- Pasang Railway Volume untuk mempertahankan memory setelah redeploy.

## 2. Environment variables utama

Atur di Railway → Service → Variables.

| Variable | Wajib | Keterangan |
|---|---:|---|
| `TOKEN` | Ya | Bot token Discord. Jangan pernah memasukkannya ke source code/log. |
| `GEMINI_API_KEY` | Untuk AI | API key Google AI Studio / Gemini API. |
| `GEMINI_MODELS` | Tidak | Urutan prioritas model, dipisahkan koma. |
| `GUILD_ID` | Ya | ID server Discord. |
| `CHANNEL_ID` | Ya | ID voice channel yang dijaga 24/7. |
| `MAIN_CHAT_CHANNEL_ID` | Untuk autonomous chat | ID channel teks tempat Cels boleh memulai obrolan. |
| `QA_CHANNEL_ID` | Tidak | Jika diisi, chat reaktif hanya aktif di channel ini dan main chat. |
| `OWNER_ID` | Tidak | ID Discord pemilik bot; kode memiliki default lama jika tidak diisi. |
| `CELS_MEMORY_PATH` | Disarankan | Path file memory persisten, misalnya `/data/cels-memory.json`. |

Nilai `GEMINI_MODELS` yang direkomendasikan:

```env
GEMINI_MODELS=gemini-3.5-flash,gemini-3.8-flash,gemini-3.5-flash-lite,gemini-3.1-flash-lite
```

Keempat model tersebut tercantum pada dokumentasi resmi Gemini API saat pengerjaan ini dilakukan. Ketersediaan aktual tetap bergantung pada project, region, kuota, dan akses API key.

## 3. Railway Volume dan memory

1. Buka service Railway → **Volumes** → tambahkan volume.
2. Mount path dapat berupa `/data`.
3. Atur:

```env
CELS_MEMORY_PATH=/data/cels-memory.json
```

Jika `CELS_MEMORY_PATH` tidak diisi tetapi Railway menyediakan `RAILWAY_VOLUME_MOUNT_PATH`, bot otomatis menyimpan `cels-memory.json` di volume tersebut. Tanpa volume, file lokal dapat hilang saat redeploy.

File lama `cels-memory.json` tetap dibaca. Format lama yang memiliki `messages`, `topics`, `worldNotes`, `userFacts`, `userSessions`, `autonomy`, dan `settings` tetap dipertahankan saat migrasi/penyimpanan.

## 4. Environment variable lama yang tetap didukung

Selain variable utama di atas, konfigurasi lama tetap dapat digunakan:

- `CELS_ADMIN_ROLE_ID`
- `AUTONOMOUS_CHAT_ENABLED`
- `AUTONOMOUS_CHAT_TICK_SECONDS`
- `AUTONOMOUS_CHAT_MIN_INTERVAL`
- `AUTONOMOUS_CHAT_MAX_PER_HOUR`
- `AUTONOMOUS_CHAT_MAX_PER_DAY`
- `AUTONOMOUS_CHAT_QUIET_MINUTES`
- `AUTONOMOUS_CHAT_FORCE_QUIET_MINUTES`
- `AUTONOMOUS_CHAT_STARTER_CHANCE`
- `AUTONOMOUS_CHAT_STARTER_ATTEMPT_MINUTES`
- `AUTONOMOUS_CHAT_UNANSWERED_HOURS`
- `AUTONOMOUS_CHAT_EVAL_COOLDOWN`
- `AUTONOMOUS_CHAT_MAX_EVALS_PER_HOUR`
- `AUTONOMOUS_CHAT_FORCE_AFTER_SILENT`
- `AUTONOMOUS_CHAT_SILENT_BACKOFF_MINUTES`
- `AUTONOMOUS_CHAT_BOOT_GRACE_MINUTES`
- `AUTONOMOUS_CHAT_SLEEP_FROM_HOUR`
- `AUTONOMOUS_CHAT_SLEEP_TO_HOUR`
- `AI_TIMEOUT_MS`
- `CELS_TEST`

Jangan menghapus variable lama yang masih dipakai deployment atau konfigurasi server.

## 5. Perilaku Gemini failover

- Model dicoba sesuai urutan `GEMINI_MODELS`, maksimal empat model unik dalam satu request.
- Timeout, respons kosong, gangguan jaringan, HTTP 429, 5xx, dan 404 akan mencoba model berikutnya.
- HTTP 404 membuat model masuk cooldown sementara (30 menit).
- HTTP 429 memberi cooldown model (60 detik); timeout/gangguan sementara juga diberi cooldown singkat.
- HTTP 401/403 menghentikan fallback karena biasanya menunjukkan API key atau izin project bermasalah.
- Setelah semua kandidat gagal, provider menerapkan backoff global bertahap agar tidak terus menghabiskan request.
- Cooldown model dilewati pada request berikutnya; model tidak diulang dalam request yang sama.

## 6. Diagnostik

- Log Railway mencatat model yang gagal, HTTP status, dan latency tanpa mencetak API key atau token.
- `/celsadmin` menampilkan status Gemini, model aktif/terakhir sukses, jumlah fallback, jumlah sukses/gagal per model, status cooldown, dan diagnostik autonomous.
- `/celsadmin aksi:tes_ai` melakukan probe model satu per satu.
- `Unknown Interaction` yang sudah kedaluwarsa tidak akan dibalas ulang; error selain itu tetap dicatat.

## 7. Deploy dan verifikasi

1. Commit/push ketiga file: `index.js`, `package.json`, dan `README.md`.
2. Deploy melalui Railway.
3. Periksa Railway Deploy Logs untuk pesan startup, Discord login, dan hasil probe Gemini.
4. Jalankan `/celsadmin aksi:tes_ai` untuk tes model.
5. Jalankan `/celsadmin aksi:status` atau `/celsadmin` untuk melihat diagnostik.
6. Pastikan bot masuk voice channel dan uji chat reaktif serta autonomous di channel yang benar.

## 8. Batas pengujian

Pemeriksaan sintaks Node.js dapat dilakukan secara lokal. Koneksi Discord dan Gemini memerlukan token/API key, izin Discord, kuota Gemini, serta environment Railway yang sebenarnya; hal-hal tersebut tidak dapat dianggap berhasil hanya dari pemeriksaan file.
