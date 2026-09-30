# Memakai AI agent

Dokumentasi ini juga tersedia dalam format yang bisa dibaca langsung oleh asisten coding
AI seperti Claude Code, Cursor, Codex, atau ChatGPT. Dengan begitu, agent menulis kode
dengan API SDK yang sebenarnya, bukan menebak nama fungsi.

| Yang Anda butuhkan | Pakai |
|---|---|
| Agent coding yang paham SDK di setiap sesi | [Agent skill](#agent-skill) |
| Seluruh dokumentasi sebagai konteks | [`llms.txt`](#llmstxt) |
| Bertanya tentang satu halaman | [Menu Copy page](#menu-copy-page) |

## Agent skill

`SKILL.md` adalah ringkasan SDK untuk agent: pilihan artifact, cara instalasi, cara
merakit SDK, aturan yang mencegah bug paling umum, dan perbedaan di Swift. Agent
memuatnya otomatis saat Anda bekerja dengan kode yang memakai `com.weekendinc.aipos`.

Untuk **Claude Code**, simpan di folder skill proyek Anda:

```bash
mkdir -p .claude/skills/aipos-sdk
curl -fsSL https://dev-mobile-wknd.github.io/aipos-sdk-docs/skills/aipos-sdk/SKILL.md \
  -o .claude/skills/aipos-sdk/SKILL.md
```

Commit folder `.claude/skills/` agar seluruh tim memakai skill yang sama. Untuk memakainya
di semua proyek Anda, simpan di `~/.claude/skills/aipos-sdk/SKILL.md`.

Untuk agent lain yang belum mendukung skill, tempelkan isi file tersebut ke berkas
instruksinya — misalnya `AGENTS.md` atau aturan proyek Cursor.

:::warning[Unduh ulang setiap kali memperbarui SDK]
Skill ditulis untuk versi SDK tertentu (saat ini 0.1.1). Setelah menaikkan versi SDK,
jalankan lagi perintah `curl` di atas agar agent tidak memakai API versi lama.
:::

## llms.txt

Seluruh dokumentasi tersedia sebagai Markdown biasa mengikuti standar
[llms.txt](https://llmstxt.org):

| Alamat | Isi |
|---|---|
| [`/llms.txt`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/llms.txt) | Daftar semua halaman beserta ringkasan satu kalimat |
| [`/llms-full.txt`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/llms-full.txt) | Seluruh dokumentasi dalam satu file |
| `<alamat halaman>.md` | Satu halaman sebagai Markdown, misalnya [`/guide/installation.md`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/installation.md) |

Berikan alamat `llms.txt` ke agent yang bisa membuka web, lalu minta ia membaca halaman
yang relevan. Untuk agent tanpa akses web, unggah `llms-full.txt` sebagai lampiran.

Versi Bahasa Indonesia tersedia di bawah `/id/`, misalnya
[`/id/llms.txt`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/llms.txt).

## Menu Copy page

Setiap halaman dokumentasi punya tombol **Copy page** di kanan atas judul:

| Aksi | Hasil |
|---|---|
| **Copy page** | Menyalin halaman sebagai Markdown, siap ditempel ke chat AI |
| **View as Markdown** | Membuka versi Markdown halaman di tab baru |
| **Open in ChatGPT** | Membuka ChatGPT dengan prompt untuk membaca halaman ini |
| **Open in Claude** | Membuka Claude dengan prompt untuk membaca halaman ini |

## Tips

- **Sebutkan versi SDK** yang Anda pakai. Semua artifact `com.weekendinc.aipos:*` harus
  berada di versi yang sama.
- **Jangan menempelkan kunci rahasia** — `llmApiKey`, `apiKey`, atau password pembayaran
  online — ke chat AI. Pakai placeholder dan isi nilainya di `local.properties`.
- **Periksa kode pembayaran** yang dihasilkan agent terhadap
  [Checklist produksi](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guides/production-checklist) sebelum dirilis.
