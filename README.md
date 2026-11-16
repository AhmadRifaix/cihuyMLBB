# cihuyMLBB
1. Jaringan (VPC, Subnet, Internet Gateway, NAT Gateway, & Route Table)
Buka layanan VPC → Create VPC (pilih opsi VPC only):

Name tag: mbg-vpc

IPv4 CIDR block: 10.20.0.0/16

Klik Create VPC, lalu buka menu Actions → Edit VPC settings dan centang Enable DNS hostnames → klik Save.

Buka Subnets → Create subnet (pilih VPC mbg-vpc) dan buat 6 subnet berikut:

Nama Subnet	Availability Zone	IPv4 CIDR Block
mbg-pub-1a	us-east-1a	10.20.1.0/24
mbg-pub-1b	us-east-1b	10.20.2.0/24
mbg-priv-app-1a	us-east-1a	10.20.10.0/24
mbg-priv-app-1b	us-east-1b	10.20.11.0/24
mbg-priv-db-1a	us-east-1a	10.20.20.0/24
mbg-priv-db-1b	us-east-1b	10.20.21.0/24
Pada masing-masing subnet publik (mbg-pub-1a dan mbg-pub-1b), buka Actions → Edit subnet settings, centang Enable auto-assign public IPv4 address → Save.

Buka Internet gateways → Create internet gateway:

Name tag: mbg-igw → Create internet gateway → klik Actions → Attach to VPC → pilih mbg-vpc.

Buka NAT gateways → Create NAT gateway:

Name: mbg-nat-gw

Subnet: mbg-pub-1a

Connectivity type: Public → klik Allocate Elastic IP → Create NAT gateway (tunggu status menjadi Available).

Buka Route tables → Create route table dan atur rute berikut:

mbg-rt-pub (VPC mbg-vpc):

Tab Routes → Edit routes: tambahkan tujuan 0.0.0.0/0 ke target Internet Gateway (mbg-igw).

Tab Subnet associations → Edit subnet associations: centang mbg-pub-1a dan mbg-pub-1b.

mbg-rt-priv (VPC mbg-vpc):

Tab Routes → Edit routes: tambahkan tujuan 0.0.0.0/0 ke target NAT Gateway (mbg-nat-gw).

Tab Subnet associations → Edit subnet associations: centang mbg-priv-app-1a, mbg-priv-app-1b, mbg-priv-db-1a, dan mbg-priv-db-1b.

2. Security Groups (5 Layer Keamanan)
Buka VPC → Security groups → Create security group (semua di dalam mbg-vpc) dan atur Inbound rules sesuai urutan berikut (Outbound biarkan default All traffic):

mbg-sg-alb:

Inbound: HTTP (TCP 80) dari Source 0.0.0.0/0.

mbg-sg-fe:

Inbound: HTTP (TCP 80) dari Source Security Group mbg-sg-alb.

mbg-sg-be:

Inbound: HTTP (TCP 80) dari Source Security Group mbg-sg-fe.

mbg-sg-efs:

Inbound: NFS (TCP 2049) dari Source Security Group mbg-sg-fe.

mbg-sg-rds:

Inbound: MYSQL/Aurora (TCP 3306) dari Source Security Group mbg-sg-be.

3. Layanan Penyimpanan, Notifikasi, & Database (S3, SNS, EFS, RDS)
A. Amazon S3 (Penyimpanan Foto Laporan & Aduan)
Buka S3 → Create bucket:

Bucket name: mbg-uploads-<ACCOUNT_ID>-us-east-1-an (ganti <ACCOUNT_ID> dengan 12 digit ID akun AWS Anda, pastikan nama unik global).

AWS Region: us-east-1.

Block Public Access: Biarkan tetap tercentang (Block all public access).

Klik Create bucket.

B. Amazon SNS (Notifikasi Real-Time)
Buka SNS → Topics → Create topic:

Type: Standard

Name: mbg-sns-notifikasi → klik Create topic.

Salin nilai ARN topic tersebut (arn:aws:sns:us-east-1:<ACCOUNT_ID>:mbg-sns-notifikasi).

Buka tab Subscriptions → Create subscription:

Protocol: Email

Endpoint: Masukkan alamat email aktif Anda → Create subscription, lalu buka kotak masuk email Anda dan klik tautan Confirm subscription.

C. Amazon EFS (Penyimpanan Session Bersama Front End)
Buka EFS → Create file system → Customize:

Name: mbg-efs-session

VPC: mbg-vpc

Klik Next → pada bagian Mount targets, pilih subnet mbg-priv-app-1a dan mbg-priv-app-1b, lalu ganti Security Group keduanya menjadi mbg-sg-efs (hapus security group default).

Klik Next → Next → Create. Catat File system ID (contoh: fs-0854786642e478603).

D. Amazon RDS MySQL
Buka RDS → Subnet groups → Create DB subnet group:

Name: mbg-db-subnet-group

VPC: mbg-vpc

Availability Zones: us-east-1a dan us-east-1b

Subnets: pilih 10.20.20.0/24 (mbg-priv-db-1a) dan 10.20.21.0/24 (mbg-priv-db-1b) → Create.

Buka RDS → Databases → Create database:

Creation method: Standard create | Engine: MySQL

Templates: Free tier (atau Dev/Test jika Free tier tidak tersedia)

DB instance identifier: mbg-rds-mysql

Master username: admin | Master password: 12345678

DB instance class: db.t3.micro (atau db.t4g.micro)

VPC: mbg-vpc | DB subnet group: mbg-db-subnet-group | Public access: No

VPC security group: Choose existing → pilih mbg-sg-rds (hapus default)

Buka Additional configuration → Initial database name: mbg_db

Klik Create database dan tunggu hingga status berubah menjadi Available, lalu salin nilai Endpoint RDS.

4. Deploy Server Back End (mbg-ec2-be)
Buka EC2 → Instances → Launch instances:

Name: mbg-ec2-be

AMI: Amazon Linux 2023 AMI

Instance type: t3.micro (atau t3.small)

Key pair: Proceed without a key pair (kita memakai AWS Systems Manager Session Manager)

Network settings → Edit:

VPC: mbg-vpc

Subnet: mbg-priv-app-1a

Auto-assign public IP: Disable

Security group: Select existing → mbg-sg-be

Advanced details → IAM instance profile: pilih LabInstanceProfile (atau role EC2 yang memiliki akses S3, SNS, dan SSM).

Klik Launch instance, tunggu status Running, catat Private IPv4 address milik mbg-ec2-be (misal 10.20.10.248).

Pilih instance mbg-ec2-be → klik Connect → Session Manager → Connect, lalu jalankan perintah berikut satu per satu di terminal:

Bash
sudo su -
hostnamectl set-hostname mbg-ec2-be && exec bash
dnf update -y
dnf install php git httpd php-mysqlnd composer -y
systemctl enable --now httpd php-fpm
Unduh kode Back End dan instal AWS SDK PHP:

Bash
cd /var/www/html && rm -rf * .* 2>/dev/null
git clone https://github.com/ukkce2627/mbg-app-be .
COMPOSER_ALLOW_SUPERUSER=1 composer require aws/aws-sdk-php --no-interaction
Ubah konfigurasi Apache agar .htaccess aktif:

Bash
nano /etc/httpd/conf/httpd.conf
Cari bagian <Directory "/var/www/html">, ubah baris AllowOverride None menjadi AllowOverride All. Simpan dengan menekan Ctrl + O, Enter, lalu keluar dengan Ctrl + X.

Edit file konfigurasi Back End (config.php):

Bash
nano /var/www/html/config.php
Sesuaikan isinya dengan Endpoint RDS, nama bucket S3, dan ARN SNS Anda:

PHP
<?php
return [
    'db' => [
        'host' => 'ENDPOINT_RDS_ANDA.us-east-1.rds.amazonaws.com',
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
Simpan (Ctrl + O, Enter) dan keluar (Ctrl + X).

Atur hak akses folder, restart layanan web, uji endpoint /health, dan jalankan seeding database awal:

Bash
chown -R apache:apache /var/www/html
systemctl restart php-fpm httpd
curl -s http://localhost/health
echo ""
curl -X POST http://localhost/api/setup/seed -H "X-API-Key: mbg-secret-key-2024"
echo ""
(Pastikan /health menampilkan db, s3, dan sns berstatus connected, dan perintah seed membalas sukses membuat 3 SPPG & 7 akun).

5. Deploy Server Front End Sementara & Buat AMI (mbg-ami-fe)
Buka EC2 → Instances → Launch instances:

Name: mbg-ec2-fe

AMI: Amazon Linux 2023 AMI | Instance type: t3.micro

Key pair: Proceed without a key pair

Network settings → Edit:

VPC: mbg-vpc | Subnet: mbg-priv-app-1a | Auto-assign public IP: Disable

Security group: Select existing → mbg-sg-fe

Advanced details → IAM instance profile: LabInstanceProfile.

Klik Launch instance, lalu hubungkan melalui Connect → Session Manager.

Jalankan perintah instalasi paket dan mount EFS session di terminal mbg-ec2-fe (ganti fs-0854786642e478603 dengan File System ID EFS Anda):

Bash
sudo su -
hostnamectl set-hostname mbg-ec2-fe && exec bash
dnf update -y
dnf install php git httpd amazon-efs-utils python3-botocore composer cronie -y
systemctl enable --now httpd php-fpm crond

mkdir -p /mnt/efs/mbg-session
echo "fs-0854786642e478603:/ /mnt/efs/mbg-session efs _netdev,tls 0 0" >> /etc/fstab
mount -a
df -h /mnt/efs/mbg-session
chmod -R 777 /mnt/efs/mbg-session
(Pastikan hasil df -h /mnt/efs/mbg-session menunjukkan 127.0.0.1:/ ter-mount).

Unduh kode Front End dan atur config.php:

Bash
cd /var/www/html && rm -rf * .* 2>/dev/null
git clone https://github.com/ukkce2627/mbg-app-fe .
nano /var/www/html/config.php
Isi dengan Private IPv4 milik mbg-ec2-be:

PHP
<?php
return [
    'api_base_url'      => 'http://PRIVATE_IP_BE_ANDA/api',
    'api_key'           => 'mbg-secret-key-2024',
    'session_save_path' => '/mnt/efs/mbg-session',
];
Simpan (Ctrl + O, Enter), keluar (Ctrl + X), lalu jalankan:

Bash
chown -R apache:apache /var/www/html
systemctl restart php-fpm httpd
curl -I http://localhost/
(Pastikan muncul respons HTTP 200 OK atau 302 Found).

Buat AMI Golden Image Front End:

Kembali ke AWS Console → EC2 → Instances, pilih mbg-ec2-fe.

Klik Actions → Image and templates → Create image.

Image name: mbg-ami-fe → klik Create image.

Buka menu AMIs dan tunggu hingga status mbg-ami-fe berubah menjadi Available (setelah Available, instance sementara mbg-ec2-fe boleh di-Terminate).

6. Target Group, Application Load Balancer (ALB), & Auto Scaling Group (ASG)
A. Target Group
Buka EC2 → Target Groups → Create target group:

Target type: Instances | Target group name: mbg-tg-fe

Protocol: HTTP | Port: 80

VPC: mbg-vpc

Health check path: /

Pada bagian Advanced health check settings, ubah Success codes menjadi 200,302 (sangat penting karena halaman awal melakukan redirect 302 ke halaman login).

Klik Next (jangan pilih instance manual) → Create target group.

B. Application Load Balancer (ALB)
Buka EC2 → Load Balancers → Create load balancer → Application Load Balancer:

Load balancer name: mbg-alb-fe

Scheme: Internet-facing | IP address type: IPv4

VPC: mbg-vpc

Mappings: Centang us-east-1a (pilih subnet mbg-pub-1a) dan us-east-1b (pilih subnet mbg-pub-1b).

Security groups: Pilih mbg-sg-alb (hapus default).

Listeners and routing: Protocol HTTP Port 80 → Forward to mbg-tg-fe.

Klik Create load balancer dan tunggu statusnya Active.

C. Launch Template & Auto Scaling Group (ASG)
Buka EC2 → Launch Templates → Create launch template:

Launch template name: mbg-lt-fe

Application and OS Images (AMI): Tab My AMIs → pilih mbg-ami-fe.

Instance type: t3.micro

Network settings: Jangan pilih subnet (biarkan kosong agar diatur oleh ASG); pada Security groups pilih mbg-sg-fe.

Advanced details → IAM instance profile: LabInstanceProfile.

Klik Create launch template.

Buka EC2 → Auto Scaling Groups → Create Auto Scaling group:

Name: mbg-asg-fe | Launch template: mbg-lt-fe → Next.

VPC: mbg-vpc | Availability Zones and subnets: pilih mbg-priv-app-1a dan mbg-priv-app-1b → Next.

Load balancing: Pilih Attach to an existing load balancer → pilih mbg-tg-fe.

Health checks: Centang Turn on Elastic Load Balancing health checks → Next.

Group size: Desired capacity 2, Min 2, Max 4.

Scaling policies: Pilih Target tracking scaling policy → Metric type: Average CPU utilization → Target value: 60.

Klik Next → Next → Create Auto Scaling group.

7. Checklist Pengujian Akhir (Nilai 100/100)
Buka EC2 → Target Groups → mbg-tg-fe → tab Targets: pastikan kedua instance dari ASG berstatus Healthy.

Salin DNS name dari mbg-alb-fe dan buka di browser (http://<DNS-ALB>).

Uji login untuk ketiga peran menggunakan password 123:

bgn (Badan Gizi Nasional): verifikasi dashboard muncul, buka menu Data SPPG dan pastikan 3 SPPG tampil.

sppg1 (Admin SPPG): uji unggah Laporan Harian beserta foto menu makanan, lalu cek bucket S3 Anda untuk memastikan file gambar masuk dan cek email Anda untuk notifikasi dari SNS.

masy1 (Masyarakat): uji kirim Aduan / Ulasan beserta foto, lalu pastikan data masuk ke daftar aduan dan tersimpan di S3.

Uji High Availability & Session Persistence: masuk sebagai bgn di browser, lalu hentikan (Terminate) salah satu instance FE di konsol EC2, kemudian refresh halaman browser—sesi login Anda harus tetap aktif tanpa ter-logout karena tersimpan di Amazon EFS, dan ASG otomatis melahirkan instance pengganti.
