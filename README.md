Penyebab error `SSL_ERROR_BAD_CERT_DOMAIN` ini adalah sertifikat SSL yang Anda buat tidak memenuhi standar browser modern, meskipun nama domain (dalam hal ini, alamat IP) terlihat cocok.

Masalah utamanya ada dua:

1.  Sertifikat Tidak Memiliki Subject Alternative Name (SAN): Browser modern seperti Firefox dan Chrome tidak lagi menggunakan kolom Common Name (CN) untuk mencocokkan alamat. Mereka wajib menggunakan ekstensi yang disebut Subject Alternative Name (SAN). Sertifikat server Anda (server.crt) kemungkinan besar dibuat tanpa ekstensi SAN ini.
2.  CA (Certificate Authority) Tidak Dipercaya: Bahkan jika Anda memperbaiki masalah SAN, Anda akan menghadapi error berikutnya. Browser Anda (di mesin Kali) tidak mengenal siapa yang menandatangani sertifikat ini (CN=gangga). Anda perlu mengimpor sertifikat CA Anda (ca.crt) ke dalam "Trust Store" (Daftar Otoritas Terpercaya) di browser atau sistem operasi Anda.

-----

### Solusi Lengkap

Anda perlu melakukan dua hal: membuat ulang sertifikat server dengan benar (menggunakan SAN), lalu mengimpor sertifikat CA ke browser Anda.

#### 1\. Buat Ulang Sertifikat Server dengan SAN

Cara termudah untuk menambahkan SAN adalah dengan menggunakan file konfigurasi OpenSSL.

a. Buat file konfigurasi baru, misalnya `san.cnf`:

sudo nano /etc/ssl/san.cnf

b. Salin dan tempel konfigurasi berikut ke dalam file `san.cnf`:
Pastikan untuk menyesuaikan bagian [dn] jika perlu. Bagian terpenting adalah [alt_names].

[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[ req_distinguished_name ]
C = ID
ST = Bali
L = Denpasar
O = BSSN
OU = PSSN
CN = 192.169.100.39

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = 192.169.100.39

  * IP.1 = 192.169.100.39 adalah baris kunci yang mendefinisikan SAN. Jika Anda juga punya nama domain, Anda bisa menambahkan DNS.1 = namadomain.com.

c. Buat ulang Certificate Signing Request (CSR) dan Private Key:
Perintah ini akan membuat server.key (baru) dan server.csr (baru) menggunakan konfigurasi san.cnf.

# Hapus dulu file lama jika perlu
# rm server.key server.csr server.crt

# Buat key dan CSR baru dengan konfigurasi SAN
openssl req -new -nodes -newkey rsa:2048 \
-keyout /etc/ssl/server.key \
-out /etc/ssl/server.csr \
-config /etc/ssl/san.cnf

d. Tanda tangani CSR baru dengan CA Anda:
Gunakan perintah openssl x509 yang lebih sederhana.

openssl x509 -req -in /etc/ssl/server.csr \
-CA /etc/ssl/ca.crt \
-CAkey /etc/ssl/ca.key \
-CAcreateserial \
-out /etc/ssl/server.crt \
-days 365 \
-extfile /etc/ssl/san.cnf \
-extensions v3_req

  * Perhatikan tambahan -extfile dan -extensions untuk memastikan ekstensi SAN dari file san.cnf disertakan dalam sertifikat final.

e. Reload Apache:
Setelah sertifikat baru (server.crt dan server.key) dibuat, muat ulang Apache.

sudo systemctl reload apache2

-----

#### 2\. Impor Sertifikat CA ke Browser Firefox Anda

Menyalin file ca.crt ke mesin Kali Anda tidaklah cukup. Anda harus memberitahu Firefox untuk mempercayainya.

1.  Buka Settings (Pengaturan) di Firefox.
2.  Cari "certificates" dan klik tombol View Certificates... (Lihat Sertifikat...).
3.  Pilih tab Authorities (Otoritas).
4.  Klik tombol Import... (Impor...).
5.  Pilih file ca.crt yang sudah Anda salin ke mesin Kali Anda.
6.  Sebuah kotak dialog akan muncul. Centang kotak "Trust this CA to identify websites" (Percayai CA ini untuk mengidentifikasi situs web).
7.  Klik OK.

Setelah menyelesaikan kedua langkah besar di atas, tutup dan buka kembali browser Anda, lalu coba akses https://192.169.100.39 lagi. Seharusnya sekarang situs Anda akan dimuat dengan ikon gembok hijau tanpa ada error.


Buat File Konfigurasi SAN
sudo nano /etc/ssl/san.cnf
Isi file dengan konfigurasi berikut (sesuaikan IP/hostname):
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[ req_distinguished_name ]
C = ID
ST = Jambi
L = Jambi
O = BSSN
OU = PSSN
CN = 192.168.56.10

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = 192.168.56.10

Buat CSR (Certificate Signing Request) dan Private Key Baru
Hapus file lama (jika ada)
sudo rm /etc/ssl/server.key /etc/ssl/server.csr /etc/ssl/server.crt
Buat private key dan CSR baru dengan konfigurasi SAN
openssl req -new -nodes -newkey rsa:2048 \
-keyout /etc/ssl/server.key \
-out /etc/ssl/server.csr \
-config /etc/ssl/san.cnf
 

Tanda Tangani Sertifikat dengan CA
openssl x509 -req -in /etc/ssl/server.csr \
-CA /etc/ssl/ca.crt \
-CAkey /etc/ssl/ca.key \
-CAcreateserial \
-out /etc/ssl/server.crt \
-days 365 \
-extfile /etc/ssl/san.cnf \
-extensions v3_req
 

Terapkan ke Web Server
sudo systemctl reload apache2
 
Salin Sertifikat CA ke Client (Kali Linux)
Copy file ca.crt dari server ke client Kali Linux, misalnya:
scp user@192.168.56.10:/etc/ssl/ca.crt ~/Downloads/

Import CA ke Firefox di Kali

Buka Firefox → Settings.
Cari Certificates → klik View Certificates....
Pilih tab Authorities.
Klik Import..., lalu pilih file ca.crt.
Centang Trust this CA to identify websites.
Klik OK.
 
Uji Koneksi HTTPS
Coba akses di web browser
https://192.168.56.10
Jika langkah benar, browser akan menampilkan ikon gembok hijau (secure) tanpa error

