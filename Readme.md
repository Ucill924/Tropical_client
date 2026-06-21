# README USER - Tropical Client

Panduan pakai `tropical_client.py` buat transfer (send), deposit ke Temple, dan cek saldo.
Wallet lo aman: **private key GAK pernah keluar dari device lo.**

---

## 1. CARA KERJA (singkat)

Tool ini connect ke gateway Tropical (node admin) buat:
- **send**: transfer USDA/CBTC ke wallet lain
- **deposit**: kirim USDA/CBTC dari wallet lo ke Temple (buat trading)
- **balance**: cek saldo wallet lo

Gateway cuma bantu **bikin hash** (hashing) transaksi. **Tanda tangan (sign) terjadi di device lo** pakai private key lo. Private key lo gak pernah dikirim ke mana-mana.

---

## 2. PERSIAPAN

### a. Yang lo butuh
- Python 3 di device (laptop/HP via Termux)
- File `tropical_client.py`
- File `client_config.json` (config lo)
- Invite code dari admin (buat daftar)

### b. Install dependency
```bash
pip install requests pynacl colorama
```

### c. Bikin client_config.json
Taro di folder yang sama dengan `tropical_client.py`:
```json
{
  "API_URL": "https://tx-api.trpcrypto.my.id",
  "username": "namamu",
  "password": "passwordmu",
  "mode_execute": "loop_wallet",

  "PARTY_ID": "<party id wallet lo>",
  "private_key_loop": "<64 hex private key - INI RAHASIA, DI DEVICE AJA>",

  "loop_bearer": "<bearer panjang dari cantonloop>",
  "loop_bearer_pendek": "<bearer pendek dari cantonloop>",
  "ws_ticket": "<ws ticket dari cantonloop>",
  "USER_ID": "<user id loop>"
}
```

**Penjelasan field:**
- `username` / `password`: akun gateway lo (buat login + auto re-login). Pilih sendiri pas register.
- `mode_execute`: `loop_wallet` (disarankan, hemat) atau `node`.
- `PARTY_ID`: alamat wallet lo (format `xxxx::1220yyyy`).
- `private_key_loop`: private key wallet lo (64 hex). **JANGAN kasih siapa-siapa.**
- `loop_bearer` / `loop_bearer_pendek` / `ws_ticket` / `USER_ID`: token dari cantonloop (buat ambil saldo + execute via Loop).

---

## 3. ONBOARDING (daftar ke node)

### Langkah 1 - Register
```bash
python tropical_client.py register <invite_code> <username> <password>
```
Contoh:
```bash
python tropical_client.py register X3Ur80AqiHU namamu passwordku123
```
Output: `Register OK. Minta admin approve, terus login.`
-> status lo **pending**.

### Langkah 2 - Tunggu admin approve
Kasih tau admin username lo. Admin bakal:
- Boarding public key lo ke node
- Aktifin akun lo

### Langkah 3 - Login
```bash
python tropical_client.py login
```
(baca username/password dari config. Atau manual: `login <user> <pass>`)
Output: `login OK, JWT disimpan (expiry 12 jam)`.

Selesai. Sekarang bisa send/deposit.

---

## 4. COMMAND LENGKAP

### whoami - info akun aktif
```bash
python tropical_client.py whoami
```
Nampilin user/party/fingerprint dari JWT aktif + sisa waktu login.

### balance - cek saldo wallet
```bash
python tropical_client.py balance
```
Nampilin saldo USDA/CBTC di wallet Loop lo.

### send - transfer ke wallet lain
```bash
python tropical_client.py send <TOKEN> <amount> <receiver_party>
```
Contoh:
```bash
python tropical_client.py send CBTC 0.0001 deb3537ca2e7867fe479bfa5fdbfa743::1220db5cf12e...
python tropical_client.py send USDA 10 7d58af16...::1220f833...
```

### deposit - kirim ke Temple (buat trading)
```bash
python tropical_client.py deposit <amount> <TOKEN>
```
Contoh:
```bash
python tropical_client.py deposit 100 USDA
python tropical_client.py deposit 0.01 CBTC
python tropical_client.py deposit all USDA      # deposit semua
```

---

## 5. MODE EXECUTE (loop_wallet vs node)

Di config `mode_execute`:

| Mode | Cara kerja | Catatan |
|------|-----------|---------|
| `loop_wallet` | hashing di node admin, execute via cantonloop | **disarankan.** Gak makan traffic node admin |
| `node` | hashing + execute di node admin | makan traffic node admin (terbatas) |

Default pakai `loop_wallet`. Ganti di config kalau perlu.

---

## 6. KEAMANAN - PENTING

- **private_key_loop = RAHASIA.** Jangan kasih admin, jangan kirim chat, jangan commit ke git.
- Gateway cuma terima PUBLIC key lo (waktu register). Private tetap di device.
- Tiap transaksi: device lo yang sign. Server gak bisa tx tanpa tanda tangan lo.
- Kalau device hilang -> private key di situ. Pindah/amankan file config.

---

## 7. TROUBLESHOOTING

**"Belum login"** -> jalanin `login` dulu.

**"login gagal"** -> cek username/password. Kalau lupa, minta admin reset (`resetpw`).

**"JWT expired -> auto re-login OK"** -> normal, JWT 12 jam, otomatis login ulang dari config.

**"saldo kurang"** -> saldo wallet kurang dari amount yang mau dikirim. Cek `balance`.

**"no holding"** -> wallet lo gak ada token itu di Loop. Cek `balance`.

**send/deposit gagal terus** -> cek `loop_bearer_pendek` di config masih valid (kadang expired, minta token baru dari cantonloop).

---

## 8. CONTOH ALUR LENGKAP

```bash
# sekali setup
pip install requests pynacl colorama
# (taro tropical_client.py + client_config.json)

# daftar
python tropical_client.py register X3Ur80AqiHU namamu passwordku
# (tunggu admin approve)

# login
python tropical_client.py login

# pakai
python tropical_client.py whoami
python tropical_client.py balance
python tropical_client.py deposit 100 USDA
python tropical_client.py send CBTC 0.001 <wallet_tujuan>
```
