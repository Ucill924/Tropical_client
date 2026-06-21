# README BOT TRADING - Temple Bot Client

Panduan pakai `temple_bot_client.py` buat **market maker CBTC/USDA di Temple** + **auto-deposit** dari wallet ke Temple.

Bot jalan di device lo. Trading pakai akun Temple lo sendiri. Deposit lewat gateway (private key tetap di device).

---

## 1. APA YANG BOT INI LAKUKAN

1. **Trade (market maker)**: pasang order buy & sell CBTC/USDA di Temple, auto-reprice kalau harga geser, auto cancel order basi.
2. **Auto-deposit**: cek saldo wallet Loop tiap N detik. Kalau saldo lewat threshold, otomatis deposit ke Temple (biar modal trading nambah).

---

## 2. PERSIAPAN

### a. Dependency
```bash
pip install curl_cffi pyotp colorama requests pynacl
```

### b. File yang dibutuhin (1 folder)
```
temple_bot_client.py
client_config.json
Market_optimization.json
```

### c. client_config.json (LENGKAP - butuh kredensial Temple)
```json
{
  "API_URL": "https://tx-api.trpcrypto.my.id",
  "username": "namamu",
  "password": "passwordku",
  "mode_execute": "loop_wallet",

  "PARTY_ID": "<party id wallet lo>",
  "private_key_loop": "<64 hex private key - DI DEVICE AJA>",
  "loop_bearer": "<bearer panjang cantonloop>",
  "loop_bearer_pendek": "<bearer pendek cantonloop>",
  "ws_ticket": "<ws ticket cantonloop>",
  "USER_ID": "<user id loop>",

  "email_temple": "<email akun Temple lo>",
  "password_temple": "<password Temple lo>",
  "totp_secret": "<TOTP secret 2FA Temple lo>"
}
```

**Field khusus trading (selain field gateway):**
- `email_temple` / `password_temple`: login Temple lo.
- `totp_secret`: secret 2FA (TOTP) Temple. Ini yang dipakai bot generate kode OTP otomatis. Ambil pas setup 2FA Temple (string base32, bukan kode 6 digit).

### d. Market_optimization.json (setting trading)
```json
{
  "MIN_ORDER_SIZE": 0.00011,
  "CYCLE_INTERVAL": 1,
  "Limit_order": 5,
  "Jeda_limit_order": 61,
  "rate_limit_cooldown": 61,
  "rate_retry_max": 3,
  "btc_price_tolerant_to_cancel_order": 60,
  "min_deposit_usda": 1600,
  "min_deposit_cbtc": 0.026,
  "wallet_check_interval": 300
}
```

**Penjelasan setting:**
| Field | Arti |
|-------|------|
| `MIN_ORDER_SIZE` | ukuran order minimum (CBTC) |
| `CYCLE_INTERVAL` | jeda antar cek (detik) |
| `Limit_order` | maks order per window (rate limit) |
| `Jeda_limit_order` | window rate limit (detik) |
| `rate_limit_cooldown` | jeda kalau kena rate limit (detik) |
| `rate_retry_max` | maks retry kalau kena rate limit |
| `btc_price_tolerant_to_cancel_order` | toleransi geser harga sebelum reprice (USDA) |
| `min_deposit_usda` | auto-deposit kalau USDA di wallet > ini |
| `min_deposit_cbtc` | auto-deposit kalau CBTC di wallet > ini |
| `wallet_check_interval` | cek saldo wallet tiap N detik (auto-deposit) |

Kalau gak mau auto-deposit: hapus / kosongin `min_deposit_usda` + `min_deposit_cbtc`.

---

## 3. ONBOARDING (sama kayak tropical_client)

Bot ini pakai akun gateway yang sama. Kalau udah daftar via tropical_client, pakai akun itu. Kalau belum:

```bash
# daftar (butuh invite code dari admin)
python temple_bot_client.py register <invite_code> <username> <password>
# (tunggu admin approve)
# login
python temple_bot_client.py login <username> <password>
```

---

## 4. CARA JALANIN

### Trade + auto-deposit (default)
```bash
python temple_bot_client.py
```
Bot login Temple, mulai market maker, + auto-deposit jalan di background.

### Trade aja (tanpa auto-deposit)
```bash
python temple_bot_client.py trade
```

### Deposit manual (sekali jalan)
```bash
python temple_bot_client.py deposit all USDA
python temple_bot_client.py deposit 0.01 CBTC
```

### Stop bot
`Ctrl + C`

---

## 5. CARA KERJA TRADING

1. Baca orderbook Temple (best bid / best ask).
2. Cek saldo Temple (CBTC + USDA).
3. Pasang order:
   - BUY pakai USDA di best bid (dipecah jadi beberapa order kecil)
   - SELL CBTC di best ask
4. Tiap cycle: cek harga. Kalau geser lebih dari toleransi -> cancel order lama, pasang ulang.
5. Reward datang dari order yang **fill** (kena), bukan dari place/cancel.

Mode trade (env opsional): `TEMPLE_TRADE_MODE=BUY` / `SELL` / `BOTH` (default BOTH).

---

## 6. CARA KERJA AUTO-DEPOSIT

1. Tiap `wallet_check_interval` detik, cek saldo wallet Loop.
2. Kalau USDA > `min_deposit_usda` atau CBTC > `min_deposit_cbtc`:
   - Trade loop **pause** sebentar.
   - Deposit `all` token itu ke Temple (lewat gateway, mode dari `mode_execute`).
   - Trade loop lanjut.

Jadi modal di wallet otomatis masuk Temple buat nambah amunisi trading.

---

## 7. JALANIN 24/7 (biar gak mati)

### Pakai screen (Linux/VPS)
```bash
screen -S templebot
python temple_bot_client.py
# detach: Ctrl+A lalu D
# balik: screen -r templebot
```

### Penting buat 24/7
- `username` + `password` HARUS ada di config (buat auto re-login pas JWT expired tiap 12 jam).
- `totp_secret` harus bener (kalau salah, login Temple gagal).

---

## 8. KEAMANAN

- `private_key_loop`, `password_temple`, `totp_secret` = RAHASIA. Di device aja.
- Gateway gak pernah terima private key / password Temple lo. Trading langsung device -> Temple. Deposit: hashing di gateway, sign di device.

---

## 9. TROUBLESHOOTING

**"Login Temple gagal"** -> cek email/password Temple + totp_secret (harus base32 secret, bukan kode 6 digit).

**"curl_cffi gak ada"** -> `pip install curl_cffi`.

**"Quote unavailable"** -> orderbook Temple lagi kosong/error, bot nunggu.

**Order kena "rate_limit"** -> normal, bot auto-cooldown + retry. Atur `Limit_order` / `Jeda_limit_order` kalau sering.

**Auto-deposit gak jalan** -> cek `min_deposit_usda`/`min_deposit_cbtc` ada di Market_optimization.json + saldo wallet lewat threshold.

**Deposit gagal** -> cek `loop_bearer_pendek` masih valid + saldo cukup.

---

## 10. CONTOH ALUR LENGKAP

```bash
# setup
pip install curl_cffi pyotp colorama requests pynacl
# (taro temple_bot_client.py + client_config.json + Market_optimization.json)

# daftar + approve + login
python temple_bot_client.py register <code> namamu passwordku
python temple_bot_client.py login namamu passwordku

# jalanin (trade + auto-deposit)
screen -S templebot
python temple_bot_client.py
# Ctrl+A D buat detach
```
