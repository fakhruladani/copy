Pendeknya: **cara HTTPS-nya sama** (CA internal + sertifikat server dengan SAN + impor CA di klien). Karena Anda mengakses **IP 192.168.56.10** dari 2 VM (Admin & Staf) lewat Hotspot/RADIUS Mikrotik, yang berbeda hanya **prasyarat jaringan** (routing/firewall) agar koneksi TCP/443 dari subnet Hotspot bisa tembus ke web server. TLS sendiri tidak peduli Anda lewat Hotspot, selama paket sampai.

Di bawah ini saya ringkas jadi 3 bagian: (A) prasyarat jaringan Mikrotik, (B) penerbitan sertifikat server dengan SAN=IP 192.168.56.10, (C) impor CA di dua VM klien.

---

## A) Prasyarat jaringan (Mikrotik/Server) – agar 10.x (Hotspot) bisa ke 192.168.56.10:443

1. **IP & gateway server**

   * Web server: `192.168.56.10/24`
   * **Default gateway server** → IP Mikrotik di subnet 192.168.56.0/24 (misal `192.168.56.1`).
   * Di Ubuntu server (netplan), pastikan default route via 192.168.56.1.

2. **Routing Mikrotik**

   * Interface Mikrotik yang menghadap server harus punya IP di `192.168.56.0/24` (misal `192.168.56.1/24`).
   * Route ke 192.168.56.0/24 akan **connected** otomatis; tidak perlu static route tambahan.

3. **Firewall Mikrotik (forward)**
   Izinkan trafik dari subnet Hotspot ke server:443. Contoh (sesuaikan nama interface/subnet):

   ```
   /ip firewall filter
   add chain=forward action=accept src-address=<subnet_hotspot>/24 dst-address=192.168.56.10 protocol=tcp dst-port=443 comment="Hotspot -> WebServer HTTPS"
   ```

   Pastikan rule **accept established,related** ada di atas, dan **drop** di paling bawah.

4. **NAT**

   * Umumnya ada `masquerade` untuk ke internet (out-interface=WAN). Trafik ke 192.168.56.0/24 tidak perlu NAT.
   * Jika Anda punya NAT yang terlalu luas, buat **bypass** agar tidak NAT ke server internal:

     ```
     /ip firewall nat
     add chain=srcnat src-address=<subnet_hotspot>/24 dst-address=192.168.56.0/24 action=accept comment="No-NAT to internal server"
     ```
   * Rule ini diletakkan **di atas** rule masquerade umum.

Jika 4 poin di atas benar, setelah login Hotspot, VM Admin & Staf harus bisa `ping 192.168.56.10` dan `telnet 192.168.56.10 443` atau `curl https://192.168.56.10` (nanti masih warning cert kalau CA belum di-trust).

---

## B) Terbitkan sertifikat server (SAN = IP 192.168.56.10)

> Sama seperti langkah “CA-only” sebelumnya, hanya **CN/SAN** yang Anda isi menjadi 192.168.56.10 (dan/atau DNS internal jika ada).

1. **(Jika belum ada) Root CA internal**

```bash
sudo mkdir -p /root/myca/{certs,crl,private,newcerts} && sudo chmod 700 /root/myca/private
sudo touch /root/myca/index.txt && echo 1000 | sudo tee /root/myca/serial >/dev/null
sudo openssl genrsa -out /root/myca/private/ca.key 4096 && sudo chmod 600 /root/myca/private/ca.key
sudo openssl req -x509 -new -nodes -sha256 -days 3650 \
  -key /root/myca/private/ca.key \
  -subj "/C=ID/ST=Jawa/L=Jakarta/O=MyOrg/OU=IT Security/CN=My Lab Root CA" \
  -out /root/myca/certs/ca.crt
```

2. **SAN untuk IP 192.168.56.10 (tambahkan DNS.* bila pakai nama host)*\*

```bash
sudo tee /etc/ssl/san.cnf >/dev/null <<'EOF'
[ req ]
default_md = sha256
prompt = no
distinguished_name = req_dn
req_extensions = v3_req

[ req_dn ]
C = ID
ST = Jawa
L = Jakarta
O = MyOrg
OU = IT
CN = 192.168.56.10

[ v3_req ]
subjectAltName = @alt_names
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ alt_names ]
IP.1 = 192.168.56.10
# DNS.1 = web.rs.local
EOF
```

3. **CSR + key (pakai key lama boleh, atau buat baru)**

```bash
# Opsi A: pakai key yang sudah ada:
# sudo openssl req -new -key /etc/ssl/private/server.key -out /etc/ssl/server.csr -config /etc/ssl/san.cnf

# Opsi B: buat key baru:
sudo openssl req -new -nodes -newkey rsa:2048 \
  -keyout /etc/ssl/private/server.key \
  -out /etc/ssl/server.csr \
  -config /etc/ssl/san.cnf
sudo chmod 600 /etc/ssl/private/server.key
```

4. **Tandatangani dengan CA**

```bash
sudo openssl x509 -req -in /etc/ssl/server.csr \
  -CA /root/myca/certs/ca.crt -CAkey /root/myca/private/ca.key \
  -CAcreateserial -out /etc/ssl/certs/server.crt \
  -days 365 -sha256 -extfile /etc/ssl/san.cnf -extensions v3_req

# Verifikasi SAN benar:
openssl x509 -in /etc/ssl/certs/server.crt -noout -text | sed -n '/Subject Alternative Name/,+3p'
```

5. **Pasang ke web server (Apache/Nginx yang sudah berjalan) lalu reload**

```bash
# Apache:
sudo systemctl reload apache2
# Nginx:
# sudo nginx -t && sudo systemctl reload nginx
```

> Pastikan vhost 443 menunjuk ke:
>
> * `SSLCertificateFile      /etc/ssl/certs/server.crt`
> * `SSLCertificateKeyFile   /etc/ssl/private/server.key`

---

## C) Impor **CA** ke dua VM klien (Admin & Staf)

> Yang diimpor ke klien adalah **ca.crt**, bukan server.crt.

### Linux (Ubuntu/Kali/DSB) — trust store OS (juga dipakai Chrome/Chromium, curl)

```bash
sudo cp /root/myca/certs/ca.crt /usr/local/share/ca-certificates/my-internal-ca.crt
sudo update-ca-certificates
```

### Firefox

* Cara cepat: `about:config` → `security.enterprise_roots.enabled` = **true** → restart.
* Atau GUI: Settings → Privacy & Security → Certificates → **View Certificates…** → **Authorities** → **Import…** → pilih `ca.crt` → centang **Trust this CA to identify websites** → OK → restart.

> Lakukan langkah ini **di VM Admin dan VM Staf**.

---

## Uji dari VM Admin/Staf

```bash
curl -I https://192.168.56.10
# atau
openssl s_client -connect 192.168.56.10:443 -servername 192.168.56.10 </dev/null \
| openssl x509 -noout -subject -issuer -ext subjectAltName
```

Di browser, buka `https://192.168.56.10` → harus muncul gembok “Secure” **(tanpa peringatan)**.

---

### Ringkasan

* **HTTPS-nya: sama** → sertifikat server valid (punya **SAN=192.168.56.10**) + **CA di-trust** oleh VM Admin & Staf.
* **Jaringan**: pastikan **setelah login Hotspot**, Mikrotik **mengizinkan** TCP/443 dari subnet Hotspot ke `192.168.56.10`, server **reply** via Mikrotik (gateway benar), NAT tidak menghalangi trafik internal.

Jika Anda berikan **subnet Hotspot** dan **IP Mikrotik** per interface, saya bisa tuliskan **rule Mikrotik** yang presisi (filter + NAT) agar akses `https://192.168.56.10` mulus.
