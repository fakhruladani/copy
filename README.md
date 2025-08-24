1) Buat CA (Certificate Authority) sendiri
Diperlukan karena Anda akan menandatangani sertifikat server dengan CA ini. Tanpa CA, perintah signing akan gagal membuka /etc/ssl/ca.crt.
1.1 Buat private key CA
sudo openssl genrsa -out /etc/ssl/ca.key 4096
1.2 Buat sertifikat CA (self-signed)
sudo openssl req -x509 -new -sha256 -days 3650 \
  -key /etc/ssl/ca.key \
  -subj "/C=ID/ST=Bali/L=Denpasar/O=BSSN/OU=PSSN/CN=My-Lab-CA" \
  -out /etc/ssl/ca.crt
2) Siapkan file SAN untuk server (wajib agar lolos browser modern)
Buat san.cnf (isi non-interaktif; fokus pada [alt_names]):
sudo tee /etc/ssl/san.cnf >/dev/null <<'EOF'
[ req ]
distinguished_name = req_distinguished_name
req_extensions     = v3_req
prompt             = no

[ req_distinguished_name ]
C  = ID
ST = Bali
L  = Denpasar
O  = BSSN
OU = PSSN
CN = 192.168.56.13

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = 192.168.56.13
# Jika punya domain:
# DNS.1 = contoh.domain.internal
EOF
Ubah semua 192.168.56.13 menjadi IP server Anda.
3) Buat key & CSR server menggunakan SAN
sudo openssl req -new -nodes -newkey rsa:2048 \
  -keyout /etc/ssl/server.key \
  -out /etc/ssl/server.csr \
  -config /etc/ssl/san.cnf
4) Tanda tangani CSR dengan CA Anda (sertifikat final punya SAN)
sudo openssl x509 -req -in /etc/ssl/server.csr \
  -CA /etc/ssl/ca.crt -CAkey /etc/ssl/ca.key -CAcreateserial \
  -out /etc/ssl/server.crt -days 365 \
  -extfile /etc/ssl/san.cnf -extensions v3_req
5) Verifikasi bahwa SAN benar-benar ada
openssl x509 -in /etc/ssl/server.crt -noout -text | grep -A1 "Subject Alternative Name"
Anda harus melihat IP Address:192.168.56.13 (atau DNS jika ditambahkan).
6) Pasang ke web server
Apache:
# contoh VirtualHost (sesuaikan path conf Anda)
# aktifkan modul ssl bila belum
sudo a2enmod ssl
# pasang path cert & key di vhost :443 Anda
# SSLCertificateFile      /etc/ssl/server.crt
# SSLCertificateKeyFile   /etc/ssl/server.key
sudo systemctl reload apache2
Nginx (opsional jika pakai Nginx):
# di server block :443
# ssl_certificate     /etc/ssl/server.crt;
# ssl_certificate_key /etc/ssl/server.key;
sudo nginx -t && sudo systemctl reload nginx
7) Buat browser mempercayai CA Anda (agar tidak ada “CA tidak dipercaya”)
Anda punya dua opsi—(A) trust di sistem dan/atau (B) trust khusus di Firefox.
A. Trust CA di OS (Debian/Ubuntu)
sudo cp /etc/ssl/ca.crt /usr/local/share/ca-certificates/myca.crt
sudo update-ca-certificates
Ini menambahkan CA ke trust store sistem. Aplikasi yang mengikuti trust store OS (curl, docker, beberapa browser berbasis Chromium) akan mempercayai sertifikat server Anda.
B. Trust CA di Firefox (punya store sendiri)
1.	Buka Settings → Privacy & Security → Certificates.
2.	Klik View Certificates… → Authorities → Import….
3.	Pilih file ca.crt.
4.	Centang “Trust this CA to identify websites.”
5.	OK, lalu restart Firefox.
8) Uji akses
Akses https://192.168.56.13 (atau IP/domain Anda).
Jika konfigurasi benar:
•	Tidak ada lagi SSL_ERROR_BAD_CERT_DOMAIN (karena SAN sudah ada).
•	Tidak ada lagi peringatan “CA tidak dipercaya” (karena CA di-import).
