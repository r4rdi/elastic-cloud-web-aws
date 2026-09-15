<div align="center">

# 🚀 AWS — VPC & EC2 Docker

Dokumentasi komprehensif mengenai konfigurasi infrastruktur AWS (VPC, Subnets, Route Tables, Internet Gateway, NAT Gateway, Elastic IP, Security Groups), peluncuran EC2 Instance, koneksi SSH CLI, deployment Container Docker dari GitHub Container Registry (GHCR), serta konfigurasi Nginx Reverse Proxy dengan SSL/TLS (HTTPS).

</div>

---

## 📌 Arsitektur & Spesifikasi Sistem

* **Cloud Provider:** Amazon Web Services (AWS Academy / AWS Labs)
* **VPC CIDR:** `10.0.0.0/16` (`wordpress-vpc`)
* **Subnets:**
  * Public Subnets: `10.0.0.0/20` (`us-east-1a`), `10.0.16.0/20` (`us-east-1b`)
  * Private Subnets: `10.0.128.0/20` (`us-east-1a`), `10.0.144.0/20` (`us-east-1b`)
* **Compute Engine:** AWS EC2 Instance (`t3.micro` / `t2.micro`, Amazon Linux 2023 kernel-6.81)
* **Container Engine:** Docker Engine
* **Docker Image:** `ghcr.io/kaound3rage/elastic-cloud-test:e573523fabe455304654ca15da78bcd198c818cf`
* **Web Server / Reverse Proxy:** Nginx with OpenSSL (Self-Signed Certificate)
* **Protocols & Ports:**
  * SSH (Port 22)
  * HTTP (Port 80 -> Auto Redirect to HTTPS)
  * HTTPS (Port 443 -> Reverse Proxy to Container Port 8080)

---

## 🛠️ Langkah-Langkah Konfigurasi & Deployment

### Langkah 1: Membuat Jaringan AWS VPC & Networking Component

1. Buka konsol **AWS Management Console > VPC**.
2. Klik **Create VPC** dan tentukan parameter berikut:
   * **Resources to create:** `VPC and more`
   * **Name tag auto-generation:** Centang *Auto-generate*, isi prefix: `wordpress`
   * **IPv4 CIDR block:** `10.0.0.0/16`
   * **Number of Availability Zones (AZs):** `2` (`us-east-1a` dan `us-east-1b`)
   * **Number of Public Subnets:** `2`
   * **Number of Private Subnets:** `2`
   * **NAT Gateways:** `Zonal (In 1 AZ)`
   * **VPC Endpoints:** `None`

---

### Langkah 2: Konfigurasi Security Group

1. Masuk ke **EC2 Console > Security Groups**.
2. Klik **Create security group**.
3. Isi rincian dasar:
   * **Security group name:** `wordpress-sg`
   * **Description:** Security Group for WordPress Web Server and SSH
   * **VPC:** Pilih `wordpress-vpc`
4. Tambahkan **Inbound Rules**:
   | Type | Protocol | Port Range | Source | Description |
   | :--- | :--- | :--- | :--- | :--- |
   | SSH | TCP | 22 | `0.0.0.0/0` | Akses SSH CLI |
   | HTTP | TCP | 80 | `0.0.0.0/0` | Web Traffic HTTP |
   | HTTPS | TCP | 443 | `0.0.0.0/0` | Web Traffic HTTPS |
5. Klik **Create security group**.

---

### Langkah 3: Meluncurkan AWS EC2 Instance

1. Buka **EC2 Console > Instances > Launch instances**.
2. Set konfigurasi instance:
   * **Name:** `wordpress-server`
   * **AMI:** Amazon Linux 2023 AMI
   * **Instance type:** `t3.micro` atau `t2.micro`
   * **Key pair:** Pilih key pair default dari AWS Labs (`labsuser` / `vockey`)
3. Pada bagian **Network settings** (klik *Edit*):
   * **VPC:** `wordpress-vpc`
   * **Subnet:** `wordpress-subnet-public1-us-east-1a`
   * **Auto-assign public IP:** `Enable`
   * **Select existing security group:** `wordpress-sg`
4. Klik **Launch instance**.
5. Catat **Public IPv4 Address** yang didapatkan (Contoh: `54.167.50.253`).

---

### Langkah 4: Menghubungkan SSH CLI dari Terminal Windows

#### A. Menggunakan Git Bash (Disarankan)
```bash
# Pindah ke direktori tempat file labsuser.pem diunduh
cd ~/Downloads

# Mengubah hak akses file kunci privat
chmod 400 labsuser.pem

# Hubungkan ke EC2 Server via SSH
ssh -i labsuser.pem ec2-user@<PUBLIC_IP_EC2>
```

#### B. Menggunakan Windows PowerShell / Command Prompt (CMD)
```powershell
# Pindah ke direktori Downloads
cd Downloads

# Jika muncul error "UNPROTECTED PRIVATE KEY FILE", atur permission file di PowerShell:
icacls.exe labsuser.pem /reset
icacls.exe labsuser.pem /grant:r "$($env:USERNAME):(R)" /inheritance:r

# Eksekusi koneksi SSH
ssh -i labsuser.pem ec2-user@<PUBLIC_IP_EC2>
```

---

### Langkah 5: Instalasi & Konfigurasi Docker Engine

Setelah berhasil SSH ke dalam instance EC2, jalankan perintah berikut:

```bash
# Update sistem paket
sudo dnf update -y

# Install Docker Engine
sudo dnf install docker -y

# Jalankan dan aktifkan service Docker saat boot
sudo systemctl start docker
sudo systemctl enable docker

# Tambahkan ec2-user ke grup docker agar tidak perlu 'sudo' saat menjalankan perintah docker
sudo usermod -aG docker ec2-user
newgrp docker

# Verifikasi instalasi Docker
docker ps
```

---

### Langkah 6: Pull Image & Deployment Container Web

Jalankan container dari GitHub Container Registry (GHCR) dan ekspos ke port internal `8080`:

```bash
# Pull image dari GHCR
docker pull ghcr.io/kaound3rage/elastic-cloud-test:e573523fabe455304654ca15da78bcd198c818cf

# Jalankan container di port internal 8080
docker run -d   -p 8080:80   --name wordpress-app   --restart always   ghcr.io/kaound3rage/elastic-cloud-test:e573523fabe455304654ca15da78bcd198c818cf

# Verifikasi status container
docker ps
```

---

### Langkah 7: Instalasi Nginx & Konfigurasi SSL/TLS (Reverse Proxy)

Gunakan Nginx untuk menangani sertifikat SSL/TLS dan melakukan proxying traffic dari port 443 ke container (port 8080).

#### A. Instalasi Nginx dan OpenSSL
```bash
sudo dnf install nginx openssl -y
```

#### B. Membuat Sertifikat SSL Self-Signed
```bash
sudo mkdir -p /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout /etc/nginx/ssl/nginx.key   -out /etc/nginx/ssl/nginx.crt   -subj "/C=ID/ST=Jakarta/L=Jakarta/O=AWSLab/CN=<PUBLIC_IP_EC2>"
```

#### C. Konfigurasi Nginx Server Block
Buat file konfigurasi Nginx baru:
```bash
sudo nano /etc/nginx/conf.d/wordpress-ssl.conf
```

Isikan konfigurasi berikut:
```nginx
server {
    listen 80;
    server_name _;
    # Redirect semua trafik HTTP ke HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;

    # Jalur Sertifikat SSL
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

#### D. Uji dan Aktifkan Nginx
```bash
# Uji sintaksis konfigurasi Nginx
sudo nginx -t

# Restart dan aktifkan Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

---

## 🚀 Pengujian & Verifikasi Output

1. Buka web browser (Google Chrome / Microsoft Edge / Mozilla Firefox).
2. Akses situs menggunakan protokol HTTPS:
   ```text
   https://<PUBLIC_IP_EC2>
   ```
3. Browser akan menampilkan peringatan keamanan karena menggunakan Self-Signed Certificate. Klik **Advanced** > **Proceed to <PUBLIC_IP_EC2> (unsafe)**.
4. Tampilan UI Web aplikasi dari repository GHCR akan berhasil dimuat dengan enkripsi HTTPS yang aktif!

---

## 📄 Lisensi & Kontribusi

Proyek ini dibuat untuk keperluan tugas/praktikum AWS Academy & AWS Labs. Bebas dikembangkan dan disesuaikan untuk kebutuhan deployment cloud infrastructure berbasis Docker dan Nginx.

