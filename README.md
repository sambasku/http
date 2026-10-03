<p align="center">
  <img src="logo.png" alt="SambasKu" width="320" />
</p>

# SambasKu HTTP

Koleksi [Bruno](https://usebruno.com) untuk menguji fungsional API
**Kamus Digital Sambas-Indonesia**. Disimpan sebagai plain file `.bru`
di git (bukan Postman export).

## Cara pakai

1. Install [Bruno](https://usebruno.com) (CLI: `brew install bruno` /
   GUI desktop)
2. Buka folder ini sebagai collection (File → Open Collection)
3. Pilih environment **local** (API di `http://localhost:3000`)
   atau salah satu environment produksi lokal (lihat di bawah)
4. Jalankan berurutan: `Register` → `Login` → request lain
   (Login menyimpan `access_token` + `refresh_token` sebagai collection
   variable untuk request berikutnya)

CLI:

```bash
cd http
npx @usebruno/cli run --env local auth/
```

Butuh seed user (admin / contributor / reviewer): `pnpm seed` di repo
[sambasku-api](https://github.com/iamutaki/sambasku-api). Kredensial non-secret
ada di `environments/local.bru`.

## Sample response

Request kunci punya blok `docs { }` (markdown + contoh JSON). Di Bruno
GUI: buka request → tab **Docs**.

Sumber kanonik semua kasus: blok `docs { }` pada file `.bru`
di collection ini.

## Struktur

```text
auth/                     # register, login (web/mobile/google/facebook), OTP, refresh, logout, hapus akun
language/                 # languages + dialects (var sambas_language_id, …)
category/                 # categories
word/                     # CRUD admin, search, media (pronounce / gambar / contoh), WOTD, report
word-suggestion/          # suggest-edit + change-history + antrean admin
word-report/              # laporan entri + resolve admin (flag-violent-image)
image/                    # upload-token ImageKit + POST /images (GitHub sambasku/images)
users/                    # profil publik, activity, avatar
contribution/             # antrean review: list, detail, approve, reject, correct + my detail
search-miss/              # pencarian kosong + dismiss admin
bookmark/                 # toggle + daftar bookmark
vote/                     # toggle, counts, my, history, deck + admin votes
comment/                  # publik + moderasi admin + blocklist CRUD
discussion/         # ruang diskusi publik + user + admin (approve/reject/takedown/pin)
verifier-applications/    # pengajuan + keputusan admin
share/                    # GET backgrounds + background-providers
notification/             # inbox + mark read
notification-campaign/    # admin template + campaign broadcast
bug-report/               # submit + admin resolve + upload-token
device/                   # registrasi device / FCM
lemma-definition/         # definisi lemma terkait
misc/                     # ping (canary CI/CD)
audit/                    # GET admin/audit-logs
environments/local.bru    # baseUrl + kredensial dev (NON-secret)
```

### Menjalankan seluruh koleksi (variable berantai)

WAJIB satu invocation. `bru.setVar` hanya hidup dalam satu proses;
invocation terpisah tidak berbagi variable:

```bash
cd http
npx @usebruno/cli run --env local auth/ language/ word/list-word-classes.bru word/create-word.bru bookmark/ word/ contribution/ search-miss/ category/ misc/ audit/
```

Urutan = dependensi: login → token; languages → id bahasa; word memakai
keduanya; bookmark sebelum soft-delete kata; contribution memverifikasi
hasil; audit membaca jejak.

Login dibatasi 5x / 15 menit per IP. Jangan menjalankan koleksi berulang
cepat.

## Aturan sinkronisasi (WAJIB)

Setiap penambahan/perubahan endpoint di `api/` **wajib** diikuti file
`.bru` di collection ini dalam PR yang sama:

- Endpoint baru → `nama-modul/nama-endpoint.bru` + minimal 1 `tests`
- Field request/response berubah → update body + assertion
- Endpoint dihapus → hapus file `.bru`-nya
Environment selain `local` (staging/production) **tidak** di-commit.
Kredensial sungguhan dikelola lokal lewat environment Bruno.
`.gitignore` memakai `environments/*.bru` + `!environments/local.bru`
supaya `production-deno.bru` / `production-render.bru` tidak lolos
hanya karena namanya bukan `production.bru` persis.

### Environment produksi (lokal, tidak di-commit)

Semua request memakai `{{baseUrl}}`, jadi memaksa satu tier cukup
dengan mengganti environment di dropdown Bruno (atau `--env` di CLI).
URL-nya bukan secret - hanya kredensial yang tidak boleh masuk git.
Buat file di `environments/` dengan `baseUrl` berikut, salin var lain
dari `local.bru`:

| Environment | `baseUrl` | Tier |
|---|---|---|
| `production` | `https://api.sambasku.com` | 1 - Cloudflare Worker |
| `production-deno` | `https://deno.sambasku.com` | 2 - Deno Deploy |
| `production-render` | `https://render.sambasku.com` | 3 - Render (bisa tidur) |

```bash
npx @usebruno/cli run --env production-render misc/ping.bru
```

`GET /api/v1/ping` mengembalikan `host` (dari header `Host`) supaya
bisa dipastikan request benar mendarat di tier yang dipilih. `runtime`
saja tidak cukup: tier 2 dan 3 keduanya `node`.
