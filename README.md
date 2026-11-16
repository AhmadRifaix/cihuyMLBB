Markdown# Panduan Deployment Aplikasi Program Makan Bergizi Gratis (MBG) di AWS

Dokumentasi langkah demi langkah konfigurasi infrastruktur **High Availability & Auto Scaling** menggunakan perintah **Bash CLI** secara manual (tanpa skrip otomatisasi/User Data) pada AWS Region **`us-east-1`**.

---

## Arsitektur Infrastruktur

* **VPC (`mbg-vpc`)**: `10.20.0.0/16` across 2 Availability Zones (`us-east-1a` & `us-east-1b`)
* **Load Balancing & Auto Scaling**: Application Load Balancer (`mbg-alb-fe`) + Auto Scaling Group (`mbg-asg-fe`, Min 2 / Desired 2 / Max 4, CPU Target 60%)
* **Compute**: EC2 Amazon Linux 2023 (Front End via AMI `mbg-ami-fe`, Back End `mbg-ec2-be`)
* **Database**: Amazon RDS MySQL (`mbg-rds-mysql`)
* **Shared Session Storage**: Amazon EFS (`/mnt/efs/mbg-session`)
* **Object Storage & Notification**: Amazon S3 & Amazon SNS

---

## 1. Jaringan (VPC, Subnet, IGW, NAT Gateway, & Route Table)

1. Buka layanan **VPC → Create VPC** (pilih **VPC only**):
   * **Name tag**: `mbg-vpc`
   * **IPv4 CIDR block**: `10.20.0.0/16`
   * Setelah dibuat, buka **Actions → Edit VPC settings**, centang **Enable DNS hostnames**, lalu klik **Save**.

2. Buka **Subnets → Create subnet** (pilih VPC `mbg-vpc`) dan buat 6 subnet berikut:

| Nama Subnet | Availability Zone | IPv4 CIDR Block | Tipe |
| :--- | :--- | :--- | :--- |
| `mbg-pub-1a` | `us-east-1a` | `10.20.1.0/24` | Public |
| `mbg-pub-1b` | `us-east-1b` | `10.20.2.0/24` | Public |
| `mbg-priv-app-1a` | `us-east-1a` | `10.20.10.0/24` | Private (App) |
| `mbg-priv-app-1b` | `us-east-1b` | `10.20.11.0/24` | Private (App) |
| `mbg-priv-db-1a` | `us-east-1a` | `10.20.20.0/24` | Private (DB) |
| `mbg-priv-db-1b` | `us-east-1b` | `10.20.21.0/24` | Private (DB) |

3. Pada masing-masing subnet publik (`mbg-pub-1a` dan `mbg-pub-1b`), buka **Actions → Edit subnet settings**, centang **Enable auto-assign public IPv4 address** → **Save**.

4. Buka **Internet gateways → Create internet gateway**:
   * **Name tag**: `mbg-igw` → **Create internet gateway** → **Actions → Attach to VPC** (`mbg-vpc`).

5. Buka **NAT gateways → Create NAT gateway**:
   * **Name**: `mbg-nat-gw`
   * **Subnet**: `mbg-pub-1a`
   * **Connectivity type**: Public → **Allocate Elastic IP** → **Create NAT gateway** (tunggu status *Available*).

6. Buka **Route tables → Create route table** dan atur rute berikut:
   * **`mbg-rt-pub`** (VPC `mbg-vpc`):
     * **Routes**: Tambahkan `0.0.0.0/0` → Target **Internet Gateway** (`mbg-igw`).
     * **Subnet associations**: Centang `mbg-pub-1a` dan `mbg-pub-1b`.
   * **`mbg-rt-priv`** (VPC `mbg-vpc`):
     * **Routes**: Tambahkan `0.0.0.0/0` → Target **NAT Gateway** (`mbg-nat-gw`).
     * **Subnet associations**: Centang `mbg-priv-app-1a`, `mbg-priv-app-1b`, `mbg-priv-db-1a`, dan `mbg-priv-db-1b`.

---

## 2. Security Groups (5 Layer Keamanan)

Buka **VPC → Security groups → Create security group** (semua di dalam `mbg-vpc`, Outbound biarkan default *All traffic*):

| Nama Security Group | Protocol & Port | Source (Inbound) | Fungsi |
| :--- | :--- | :--- | :--- |
| `mbg-sg-alb` | HTTP (TCP `80`) | `0.0.0.0/0` | Akses publik ke Load Balancer |
| `mbg-sg-fe` | HTTP (TCP `80`) | Security Group `mbg-sg-alb` | Trafik dari ALB ke Front End |
| `mbg-sg-be` | HTTP (TCP `80`) | Security Group `mbg-sg-fe` | Request API dari Front End ke Back End |
| `mbg-sg-efs` | NFS (TCP `2049`) | Security Group `mbg-sg-fe` | Mount EFS Session di Front End |
| `mbg-sg-rds` | MYSQL/Aurora (TCP `3306`) | Security Group `mbg-sg-be` | Koneksi database dari Back End ke RDS |

---

## 3. Layanan Penyimpanan, Notifikasi, & Database

### A. Amazon S3
1. Buka **S3 → Create bucket**:
   * **Bucket name**: `mbg-uploads-<ACCOUNT_ID>-us-east-1-an`
   * **AWS Region**: `us-east-1`
   * **Block Public Access**: Tetap centang *Block all public access* → **Create bucket**.

### B. Amazon SNS
1. Buka **SNS → Topics → Create topic**:
   * **Type**: `Standard` | **Name**: `mbg-sns-notifikasi` → **Create topic**.
   * Catat **ARN** topic (`arn:aws:sns:us-east-1:<ACCOUNT_ID>:mbg-sns-notifikasi`).
2. Buka tab **Subscriptions → Create subscription**:
   * **Protocol**: `Email` | **Endpoint**: Masukkan email aktif → **Create subscription**, lalu klik **Confirm subscription** pada email yang masuk.

### C. Amazon EFS
1. Buka **EFS → Create file system → Customize**:
   * **Name**: `mbg-efs-session` | **VPC**: `mbg-vpc`
   * Klik **Next** → pada **Mount targets**, pilih `mbg-priv-app-1a` dan `mbg-priv-app-1b`, ubah Security Group keduanya menjadi **`mbg-sg-efs`** (hapus `default`).
   * Klik **Next → Next → Create**. Catat **File system ID** (contoh: `fs-0854786642e478603`).

### D. Amazon RDS MySQL
1. Buka **RDS → Subnet groups → Create DB subnet group**:
   * **Name**: `mbg-db-subnet-group` | **VPC**: `mbg-vpc`
   * **Availability Zones**: `us-east-1a`, `us-east-1b`
   * **Subnets**: `10.20.20.0/24` (`mbg-priv-db-1a`) dan `10.20.21.0/24` (`mbg-priv-db-1b`) → **Create**.
2. Buka **RDS → Databases → Create database**:
   * **Creation method**: Standard create | **Engine**: MySQL | **Templates**: Free tier / Dev/Test
   * **DB instance identifier**: `mbg-rds-mysql`
   * **Master username**: `admin` | **Master password**: `12345678`
   * **DB instance class**: `db.t3.micro`
   * **VPC**: `mbg-vpc` | **DB subnet group**: `mbg-db-subnet-group` | **Public access**: No
   * **VPC security group**: **`mbg-sg-rds`** (hapus `default`)
   * **Additional configuration → Initial database name**: `mbg_db`
   * Klik **Create database**, tunggu hingga *Available*, lalu salin **Endpoint** RDS.

---

## 4. Konfigurasi Server Back End (`mbg-ec2-be`)

1. Buka **EC2 → Instances → Launch instances**:
   * **Name**: `mbg-ec2-be` | **AMI**: Amazon Linux 2023 | **Instance type**: `t3.micro`
   * **Key pair**: Proceed without a key pair
   * **VPC**: `mbg-vpc` | **Subnet**: `mbg-priv-app-1a` | **Auto-assign public IP**: Disable
   * **Security group**: **`mbg-sg-be`** | **IAM instance profile**: `LabInstanceProfile`
   * Klik **Launch instance** dan catat **Private IPv4 address** `mbg-ec2-be`.

2. Hubungkan ke `mbg-ec2-be` melalui **Connect → Session Manager**, lalu jalankan perintah Bash berikut:

```bash
sudo su -
hostnamectl set-hostname mbg-ec2-be && exec bash
dnf update -y
dnf install php git httpd php-mysqlnd composer -y
systemctl enable --now httpd php-fpm

cd /var/www/html && rm -rf * .* 2>/dev/null
git clone [https://github.com/ukkce2627/mbg-app-be](https://github.com/ukkce2627/mbg-app-be) .
COMPOSER_ALLOW_SUPERUSER=1 composer require aws/aws-sdk-php --no-interaction
Aktifkan .htaccess pada Apache:Bashnano /etc/httpd/conf/httpd.conf
# Pada bagian <Directory "/var/www/html">, ubah AllowOverride None menjadi AllowOverride All
# Simpan: Ctrl+O -> Enter -> Ctrl+X
Edit /var/www/html/config.php:Bashnano /var/www/html/config.php
PHP<?php
return [
    'db' => [
        'host' => '<ENDPOINT_RDS_ANDA>',
        'port' => '3306',
        'user' => 'admin',
        'pass' => '12345678',
        'name' => 'mbg_db',
    ],
    's3' => [
        'bucket' => 'mbg-uploads-<ACCOUNT_ID>-us-east-1-an',
        'region' => 'us-east-1',
    ],
    'sns' => [
        'topic_arn' => 'arn:aws:sns:us-east-1:<ACCOUNT_ID>:mbg-sns-notifikasi',
    ],
    'api_key' => 'mbg-secret-key-2024',
];
Set permission, restart web server, cek /health, dan jalankan seeding database awal:Bashchown -R apache:apache /var/www/html
systemctl restart php-fpm httpd
curl -s http://localhost/health
echo ""
curl -X POST http://localhost/api/setup/seed -H "X-API-Key: mbg-secret-key-2024"
echo ""
5. Konfigurasi Server Front End & Pembuatan AMI (mbg-ami-fe)Buka EC2 → Instances → Launch instances:Name: mbg-ec2-fe | AMI: Amazon Linux 2023 | Instance type: t3.microKey pair: Proceed without a key pairVPC: mbg-vpc | Subnet: mbg-priv-app-1a | Auto-assign public IP: DisableSecurity group: mbg-sg-fe | IAM instance profile: LabInstanceProfileHubungkan ke mbg-ec2-fe melalui Connect → Session Manager, lalu jalankan perintah Bash berikut (sesuaikan <EFS_ID>):Bashsudo su -
hostnamectl set-hostname mbg-ec2-fe && exec bash
dnf update -y
dnf install php git httpd amazon-efs-utils python3-botocore composer cronie -y
systemctl enable --now httpd php-fpm crond

mkdir -p /mnt/efs/mbg-session
echo "<EFS_ID>:/ /mnt/efs/mbg-session efs _netdev,tls 0 0" >> /etc/fstab
mount -a
df -h /mnt/efs/mbg-session
chmod -R 777 /mnt/efs/mbg-session

cd /var/www/html && rm -rf * .* 2>/dev/null
git clone [https://github.com/ukkce2627/mbg-app-fe](https://github.com/ukkce2627/mbg-app-fe) .
Edit /var/www/html/config.php (isi dengan Private IPv4 milik mbg-ec2-be):Bashnano /var/www/html/config.php
PHP<?php
return [
    'api_base_url'      => 'http://<PRIVATE_IP_BE>/api',
    'api_key'           => 'mbg-secret-key-2024',
    'session_save_path' => '/mnt/efs/mbg-session',
];
Set permission dan verifikasi respons web:Bashchown -R apache:apache /var/www/html
systemctl restart php-fpm httpd
curl -I http://localhost/
Buat Golden Image AMI:Di Console EC2, pilih mbg-ec2-fe → Actions → Image and templates → Create image.Image name: mbg-ami-fe → Create image.Tunggu di menu AMIs hingga status mbg-ami-fe menjadi Available (setelah Available, instance mbg-ec2-fe dapat di-Terminate).6. Target Group, Application Load Balancer, & Auto Scaling GroupA. Target Group (mbg-tg-fe)Target type: Instances | Protocol: HTTP | Port: 80 | VPC: mbg-vpcHealth check path: /Advanced health check settings → Success codes: 200,302B. Application Load Balancer (mbg-alb-fe)Scheme: Internet-facing | VPC: mbg-vpcMappings: us-east-1a (mbg-pub-1a) dan us-east-1b (mbg-pub-1b)Security group: mbg-sg-albListener: HTTP:80 → Forward to mbg-tg-feC. Launch Template (mbg-lt-fe) & Auto Scaling Group (mbg-asg-fe)Launch Template: AMI mbg-ami-fe, Instance type t3.micro, Security group mbg-sg-fe, IAM instance profile LabInstanceProfile.Auto Scaling Group:VPC: mbg-vpc | Subnets: mbg-priv-app-1a, mbg-priv-app-1bLoad balancing: Attach to mbg-tg-fe | Health checks: Turn on ELB health checksGroup size: Desired 2, Min 2, Max 4Scaling policy: Target tracking scaling policy → Average CPU utilization 607. Akun Pengujian & Checklist VerifikasiAkses aplikasi melalui DNS Name milik mbg-alb-fe (http://<DNS-ALB>):RoleUsernamePasswordSkenario PengujianBadan Gizi Nasional (BGN)bgn123Cek dashboard utama & daftar 3 SPPGAdmin SPPGsppg1123Upload laporan harian + foto (verifikasi masuk ke S3 & email SNS)Masyarakatmasy1123Kirim aduan/ulasan + foto (verifikasi masuk ke database & S3)Pengujian High Availability:Login sebagai bgn di browser, lalu Terminate salah satu instance FE di EC2 Console. Refresh browser untuk memastikan sesi login tetap aktif (tersimpan di EFS) dan ASG otomatis membuat instance pengganti hingga kembali menjadi 2 instance Healthy.
