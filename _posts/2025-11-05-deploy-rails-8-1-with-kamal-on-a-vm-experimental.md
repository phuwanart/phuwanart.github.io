---
layout: post
title: Deploy Rails 8.1 with Kamal on a VM (Experimental)
categories: Posts
tags:
- rails
- rails 8.1
- kamal
- docker
- deploy
date: 2025-11-05 13:55 +0700
---

หลังจาก [Rails 8.1](https://rubyonrails.org/blog/) ปล่อยออกมา ฟีเจอร์ที่รออยู่นานคือ **Registry-Free Kamal Deployments**

ปกติแล้ว Kamal ต้องการ Docker registry (Docker Hub, GHCR, registry.digitalocean.com, ฯลฯ) เป็นตัวกลาง — เครื่อง dev `docker push` ขึ้นไป แล้ว server `docker pull` ลงมารัน ปัญหาคือ Docker Hub แบบ free มี private repo ได้แค่ 1 ตัว ส่วน GHCR ก็ผูกอยู่กับ GitHub ถ้ามีหลายแอปต้องจ่ายเงินหรือเปลี่ยน provider

Kamal 2.7+ ที่มากับ Rails 8.1 แก้ปัญหานี้ด้วยการ run Docker registry container บน deploy server เลย (default ที่ `localhost:5555`) — Kamal build image บนเครื่อง dev แล้ว push ตรงไปที่ registry บน server ผ่าน SSH tunnel ไม่ต้องพึ่ง registry ภายนอกเลย

โพสต์นี้พา deploy Rails 8.1 บน VM ด้วยวิธีนี้ — ใช้ได้ทั้ง VM ที่บ้าน, DigitalOcean Droplet, Hetzner, หรือ VPS provider อื่นที่ให้เราเข้าถึง Ubuntu ผ่าน SSH

## Prerequisites

- Rails 8.1+ บนเครื่อง dev
- Ruby 3.3+
- VM/Server (Ubuntu 22.04+ แนะนำ) ที่เข้าถึงด้วย SSH ได้ พร้อม sudo access
- Docker Desktop หรือ OrbStack บนเครื่อง dev (สำหรับ build image)

## Getting Started

ในการทดลองนี้จะลอง deploy บน VM ก่อน ซึ่งน่าจะเป็นแนวทางในการ deploy บน server จริง ๆ ของเรา หรือบน cloud service ที่มีการจัดการ VPS ให้เรา อย่างเช่น [DigitalOcean](https://www.digitalocean.com/)

### Create a Server

ติดตั้ง SSH server และตั้ง firewall ให้เปิดเฉพาะ port ที่จำเป็น:

```sh
sudo apt install openssh-server ufw
sudo ufw status
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo systemctl restart ssh
sudo systemctl status ssh
```
{: file="root@1.2.3.4"}

UFW (Uncomplicated Firewall) default จะ deny ทุก inbound port — ต้อง allow `ssh` (22), `http` (80), และ `https` (443) เพราะ Kamal-proxy ต้องการ 80/443 สำหรับ serve app และ Let's Encrypt cert (ถ้าเปิดใช้ SSL)

### Create Deploy User

ห้ามใช้ root deploy โดยตรง — สร้าง user แยกชื่อ `deploy`:

```sh
sudo adduser deploy
```
{: file="root@1.2.3.4"}

เพิ่ม `deploy` เข้า `sudo` group เพื่อให้รันคำสั่ง privileged ได้ (Kamal ต้องการตอน install Docker):

```sh
sudo adduser deploy sudo
```
{: file="root@1.2.3.4"}

login เข้า deploy user:

```sh
su deploy
```
{: file="root@1.2.3.4"}

เพิ่ม group `docker` และใส่ `deploy` เข้ากลุ่ม เพื่อให้รัน `docker` ได้โดยไม่ต้อง sudo:

```sh
sudo addgroup docker
sudo usermod -aG docker $USER
```
{: file="deploy@1.2.3.4"}

> ต้อง logout แล้ว login ใหม่ (เพื่อให้สิทธิ์กลุ่ม docker มีผล)
{: .prompt-tip }

สร้าง directory สำหรับ storage ของ Rails application (จะถูก bind-mount เข้า container ทีหลัง):

```sh
mkdir blog_storage
```
{: file="deploy@1.2.3.4"}

แก้ไฟล์ sudoers ให้ `deploy` รัน sudo ได้โดยไม่ต้องใส่ password:

```sh
sudo visudo
```
{: file="deploy@1.2.3.4"}

เพิ่ม `deploy  ALL=(ALL) NOPASSWD:ALL` หลังบรรทัด `%sudo   ALL=(ALL:ALL) ALL`:

```sh
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

deploy  ALL=(ALL) NOPASSWD:ALL
```
{: file="/etc/sudoers"}

ทำไมต้อง NOPASSWD — Kamal ssh เข้าไปรัน command แบบ non-interactive ไม่มีทาง input password ระหว่าง deploy ถ้าไม่ตั้ง NOPASSWD ทุก task ที่ใช้ sudo จะค้าง

### Add SSH Key for Deploy

#### Using `ssh-copy-id`

```sh
brew install ssh-copy-id
```
{: file="Local Machine"}

```sh
ssh-copy-id deploy@1.2.3.4
```
{: file="Local Machine"}

```sh
ssh deploy@1.2.3.4
```
{: file="Local Machine"}

#### Using `authorized_keys`

อ่าน public key บนเครื่อง dev:

```sh
# for RSA SSH key
cat ~/.ssh/id_rsa.pub
```
{: file="Local Machine"}

```sh
# for ED25519 SSH key
cat ~/.ssh/id_ed25519.pub
```
{: file="Local Machine"}

บน server สร้างไฟล์ `authorized_keys` ของ deploy user:

```sh
mkdir ~/.ssh && touch ~/.ssh/authorized_keys
```
{: file="deploy@1.2.3.4"}

```sh
sudo vi ~/.ssh/authorized_keys
```
{: file="deploy@1.2.3.4"}

paste public key ลงไป:

```sh
ssh-ed25519 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX user@computer
```
{: file="~/.ssh/authorized_keys"}

ทดสอบจากเครื่อง dev:

```sh
ssh deploy@1.2.3.4
```
{: file="Local Machine"}

### Install Docker Desktop or OrbStack

บนเครื่อง dev เลือกอย่างใดอย่างหนึ่ง:

```sh
brew install --cask docker-desktop
```
{: file="Local Machine"}

หรือ (เร็วกว่า ใช้ resource น้อยกว่าบน macOS):

```sh
brew install --cask orbstack
```
{: file="Local Machine"}

### Create Rails Application

```sh
rails new blog
cd blog
rails generate controller home index
```
{: file="Local Machine"}

ตั้ง root route:

```ruby
root "home#index"
```
{: file="config/routes.rb"}

### Configuring Kamal

แก้ `config/deploy.yml`:

```diff
 # Name of your application. Used to uniquely configure containers.
 service: blog

 # Name of the container image (use your-user/app-name on external registries).
 image: blog

 # Deploy to these servers.
 servers:
   web:
-    - 192.168.0.1
+    - [ip_address ของ server]
   # job:
   #   hosts:
   #     - 192.168.0.1
   #   cmd: bin/jobs

 # Enable SSL auto certification via Let's Encrypt and allow for multiple apps on a single web server.
 # ...
 # proxy:
 #   ssl: true
 #   host: app.example.com

 # Where you keep your container images.
 registry:
   # Alternatives: hub.docker.com / registry.digitalocean.com / ghcr.io / ...
   server: localhost:5555

   # Needed for authenticated registries.
   # username: your-user

   # Always use an access token rather than real password when possible.
   # password:
   #   - KAMAL_REGISTRY_PASSWORD

 # Inject ENV variables into containers (secrets come from .kamal/secrets).
 env:
   secret:
     - RAILS_MASTER_KEY
   clear:
     SOLID_QUEUE_IN_PUMA: true

 aliases:
   console: app exec --interactive --reuse "bin/rails console"
   shell: app exec --interactive --reuse "bash"
   logs: app logs -f
   dbc: app exec --interactive --reuse "bin/rails dbconsole --include-password"

 # Use a persistent storage volume for sqlite database files and local Active Storage files.
 volumes:
-  - "blog_storage:/rails/storage"
+  - "./blog_storage:/rails/storage"

 asset_path: /rails/public/assets

 builder:
   arch: amd64

-# ssh:
-#   user: app
+ssh:
+  user: deploy
```
{: file="config/deploy.yml"}

จุดสำคัญในไฟล์นี้:

- **`registry: server: localhost:5555`** — บรรทัดที่เป็นหัวใจของ Registry-Free Deployment — บอก Kamal ให้ใช้ registry ที่รันอยู่บน deploy server (Kamal จะตั้ง container `kamal-docker-registry` ให้เองตอน `kamal setup`) Kamal push image ผ่าน SSH tunnel เข้า port 5555 บน server โดยตรง ไม่ผ่าน internet
- **`volumes: ./blog_storage:/rails/storage`** — เปลี่ยนจาก Docker named volume (`blog_storage:`) เป็น bind mount โฟลเดอร์ host (`./blog_storage`) ที่สร้างไว้ใน home ของ deploy user — backup ง่ายกว่า เห็นไฟล์จริงบน host เลย และคุม permission ได้ตรงๆ
- **`ssh: user: deploy`** — สั่ง Kamal ssh ด้วย user `deploy` ที่เพิ่งสร้าง ไม่ใช่ root
- **`env.secret: RAILS_MASTER_KEY`** — ดึงค่ามาจาก `.kamal/secrets` (ส่วนถัดไป) ไม่ใช่ฮาร์ดโค้ดใน yaml

### ตั้ง `.kamal/secrets`

ไฟล์นี้คือ bash script ที่ Kamal source ก่อน deploy เพื่อโหลด secret ลง environment ค่าจะถูกส่งเข้า container ตอน start ไม่ได้ commit ลง repo

```sh
RAILS_MASTER_KEY=$(cat config/master.key)
```
{: file=".kamal/secrets"}

ถ้าใช้ password manager (เช่น 1Password CLI) แทน:

```sh
RAILS_MASTER_KEY=$(op read "op://Vault/blog/RAILS_MASTER_KEY")
```
{: file=".kamal/secrets"}

### Setup และ Deploy

commit ทุกอย่างให้เรียบร้อย แล้วรัน:

```sh
kamal setup
```
{: file="Local Machine"}

`kamal setup` ทำงานหลายอย่างในคำสั่งเดียว:

1. ssh เข้า server เป็น `deploy` user
2. ติดตั้ง Docker บน server (ถ้ายังไม่มี)
3. ตั้ง `kamal-proxy` container (Traefik รุ่นเบาที่ Kamal เขียนเอง) ฟัง port 80/443
4. ตั้ง local Docker registry container ที่ port 5555
5. build image บนเครื่อง dev แล้ว push ผ่าน SSH tunnel ไปที่ local registry บน server
6. pull image จาก local registry แล้ว start app container
7. ผูก app container เข้ากับ kamal-proxy

หลัง setup เสร็จเปิด browser ไปที่ `http://<ip ของ server>` จะเห็นหน้า home

ครั้งต่อไปใช้:

```sh
kamal deploy
```
{: file="Local Machine"}

แค่ build + push + restart container ไม่ต้องตั้ง infra ใหม่

### Verify

ดู log ของ app:

```sh
kamal app logs -f
```
{: file="Local Machine"}

เช็ค container ที่รันบน server:

```sh
docker ps
```
{: file="deploy@1.2.3.4"}

ควรเห็น `kamal-proxy`, `kamal-docker-registry`, และ `blog-web-<version>` รันอยู่

## Troubleshooting

**`Permission denied (publickey)` ตอน Kamal ssh เข้า server**
SSH key บนเครื่อง dev ยังไม่ได้ copy ไปที่ `~/.ssh/authorized_keys` ของ deploy user ลอง `ssh deploy@<ip>` ด้วยมือก่อน ถ้าผ่านค่อย retry `kamal setup`

**`sudo: a terminal is required to read the password`**
ลืม NOPASSWD ใน `/etc/sudoers` Kamal จะค้างรอ password ที่ไม่มีทาง input เปิด visudo เพิ่มบรรทัด `deploy ALL=(ALL) NOPASSWD:ALL` แล้ว retry

**Browser เข้า `http://<ip>` แล้ว timeout**
UFW block — เช็คด้วย `sudo ufw status` ต้องมีทั้ง `80/tcp ALLOW` และ `443/tcp ALLOW`

**`kamal-proxy` start ไม่ได้ — port already in use**
มี service อื่นบน host ใช้ port 80/443 อยู่ (Apache, Nginx ที่ติดมาจาก default) สั่งหยุดและ disable ก่อน:

```sh
sudo systemctl stop apache2 nginx
sudo systemctl disable apache2 nginx
```
{: file="deploy@1.2.3.4"}

**Image push ช้ามาก**
Registry-Free Kamal push image ผ่าน SSH tunnel ซึ่งช้ากว่า push ตรงไป Docker Hub สำหรับ build ขนาดใหญ่ ลองใช้ `builder.remote` ตามตัวอย่างใน deploy.yml (build บน remote arm/amd64 server) เพื่อข้าม transfer image ข้าม internet

## Conclusion

ได้ Rails 8.1 รันบน server ด้วย Kamal โดยไม่ต้องสมัคร registry ภายนอกเลย — ทุกอย่างอยู่บน server เดียว: app, kamal-proxy, local registry

**Pattern นี้เหมาะกับ:**
- Single-server deployment (1 VM/VPS รันแอปเดียว)
- Side project ที่ไม่อยากจ่ายค่า Docker Hub Pro
- Self-host บน home server หรือ Raspberry Pi

**กลับไปใช้ external registry เมื่อ:**
- มี server หลายตัวต้อง pull image ชุดเดียวกัน
- ต้องการ rollback ไป image version เก่าๆ ที่ไม่ได้เก็บบน server ปัจจุบัน
- ใช้ CI/CD pipeline ที่ build บน GitHub Actions/GitLab CI แล้ว push ขึ้น registry กลาง

## References

- [Kamal documentation](https://kamal-deploy.org/)
- [Rails 8.1 release notes](https://rubyonrails.org/blog/)
- [Kamal Deployment: The Newest Form of Self-Torture](https://alec-c4.com/posts/2025-04-02-kamal/)
- [Rails Application Deployment on Digital Ocean](https://medium.com/@qasimali7566675/rails-application-deployment-on-digital-ocean-8483d3f10813)
- [Ultimate Guide to Server Hardening for Kamal](https://blog.cloud66.com/ultimate-guide-to-server-hardening-for-kamal)
