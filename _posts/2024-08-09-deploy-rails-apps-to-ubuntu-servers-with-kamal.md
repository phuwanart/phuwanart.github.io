---
layout: post
title: Deploy Rails apps to Ubuntu servers with Kamal
categories: Posts
tags: deploy rails ubuntu kamal docker
date: 2024-08-09 15:10 +0700
---

[Kamal](https://kamal-deploy.org/) เป็นเครื่องมือที่ใช้สำหรับ deploy web app ด้วย Docker — เทียบกับ Capistrano ที่ ssh เข้าไป pull code แล้ว restart Puma/Nginx โดยตรง Kamal จะ build image บนเครื่อง dev push ขึ้น registry แล้วให้ server pull ลงมารันใน Docker container ทั้งหมด ทำให้ environment เหมือนกันทุกที่และ rollback ง่าย

ในที่นี้เราจะลองใช้เครื่องมือนี้ deploy ลง server ของเราเอง

> โพสต์นี้เขียนสมัย **Kamal 1.x** ใช้ `.env` เก็บ secret และ Traefik เป็น reverse proxy ถ้าใช้ Kamal 2 (default ของ Rails 8+) ที่ใช้ `.kamal/secrets` กับ `kamal-proxy` ดูที่ [Deploy Rails 8.1 with Kamal on a VM](/posts/deploy-rails-8-1-with-kamal-on-a-vm-experimental/) ซึ่งครอบ Registry-Free Deployment ด้วย
{: .prompt-warning }

## Prerequisites

สำหรับเครื่อง local:

- Ruby 3.3.4
- Rails 7.2.0.rc1
- Kamal 1.8.1
- Docker Desktop (หรือ OrbStack บน macOS)
- Docker Hub account (สำหรับใช้เป็น registry)

ฝั่ง server:

- Ubuntu 22.04+
- SSH access พร้อม sudo

## Preparing Server

เริ่มจากที่เราติดตั้ง Ubuntu server หลังจากติดตั้งเสร็จให้ทำการตั้งค่า firewall[^config-firewall] ให้เรียบร้อยเสียก่อน

[^config-firewall]: [Config firewall](https://phuwanart.github.io/posts/how-to-deploy-rails-7-2-from-scratch-with-capistrano-puma-and-nginx-on-ubuntu-24-04/#setup-firewall)

จากนั้นตรวจดูว่าเราสามารถ ssh เข้าไปได้:

```sh
ssh ubuntu@SERVER_ADDRESS
exit
```

แล้วก็ทำการอนุญาตให้ login ได้โดยไม่ต้องใช้ password (Kamal ต้องการ key-based auth สำหรับ deploy แบบ non-interactive):

```sh
ssh-copy-id ubuntu@SERVER_ADDRESS
ssh ubuntu@SERVER_ADDRESS
```

อัพเดทและอัพเกรด server ให้เรียบร้อย:

```sh
sudo apt update
sudo apt upgrade -y
```

### Install Docker[^install-docker-ubuntu]

[^install-docker-ubuntu]: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

- ตั้งค่า Docker's `apt` repository:

```sh
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
```

- ติดตั้ง Docker packages ตัวล่าสุด:

```sh
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- อนุญาตให้รัน docker ได้โดยไม่ต้องใช้ sudo:

```sh
sudo usermod -a -G docker ubuntu
sudo apt install -y curl git
exit
```

ตรวจว่า docker รันอยู่มั้ยโดยการ login เข้าอีกครั้ง:

```sh
$ ssh ubuntu@SERVER_ADDRESS
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

ใน Rails application ที่เราจะสร้าง เราจะใช้ SQLite ในการทดลองนี้ เพราะฉะนั้นเราจะต้องสร้าง directory เพื่อให้ Docker มา mount ไฟล์ไว้ที่นี่ (จะถูก bind-mount เข้า container ตอน deploy):

```sh
mkdir hello-storage
sudo chmod 777 hello-storage
exit
```

> `chmod 777` เปิดสิทธิ์ให้ทุก user บน server อ่าน/เขียน/รันได้ — สะดวกตอนทดลองแต่ไม่ปลอดภัยบน production จริง ทางที่ดีกว่าคือใช้ `chmod 770` แล้วตั้ง group ให้ตรงกับ user ที่รันใน container (UID 1000 ของ Rails image default)
{: .prompt-info }

## Preparing Rails

### Create new rails application

```sh
rails new hello
cd hello
bundle lock --add-platform aarch64-linux
bin/rails g scaffold Post body:text
bin/setup
```

`bundle lock --add-platform aarch64-linux` เพิ่ม Linux ARM64 ลง `Gemfile.lock` — ถ้าเครื่อง dev เป็น Apple Silicon (M1/M2/M3) bundler default จะ resolve gem แค่สำหรับ `arm64-darwin` ตอน build image สำหรับ Linux server (ที่อาจเป็น amd64 หรือ arm64) จะ error gem ไม่ตรง platform บรรทัดนี้บอก bundler ให้เพิ่ม platform เผื่อไว้

แก้ไข routes.rb:

```ruby
Rails.application.routes.draw do
  resources :posts
  root "posts#index"
end
```
{: file="config/routes.rb"}

ปิด force SSL ก่อน เพราะยังไม่ได้ตั้ง SSL certificate ให้ Traefik — ถ้าเปิดไว้จะ redirect HTTP → HTTPS แล้ว browser โหลดไม่ขึ้น:

```ruby
config.force_ssl = false
```
{: file="config/environments/production.rb"}

เสร็จแล้วลองรันดู:

```sh
bin/rails s
open http://127.0.0.1:3000/
```

### Install Kamal

```sh
gem install kamal
kamal init --bundle
```

`kamal init --bundle` generate ไฟล์ที่จำเป็น: `config/deploy.yml`, `.kamal/hooks/`, `bin/kamal` wrapper script และเพิ่ม `kamal` ลง Gemfile ของ project (`--bundle` flag) เพื่อให้ทีมใช้ Kamal version เดียวกัน

ต่อไปก็แก้ไฟล์ `deploy.yml` ดังตัวอย่างนี้:

```yaml
service: hello
image: phuwanart/hello
servers:
  - SERVER_ADDRESS
registry:
  username: phuwanart
  password:
    - KAMAL_REGISTRY_PASSWORD
env:
  secret:
    - RAILS_MASTER_KEY
ssh:
  user: ubuntu
volumes:
  - "./hello-storage:/rails/storage"
```
{: file="config/deploy.yml"}

อธิบายแต่ละ field:

- **`service`** — ชื่อ app ใช้เป็น prefix ของ container (`hello-web-<version>`)
- **`image`** — ชื่อ image บน registry (`<docker-hub-user>/<image-name>`)
- **`servers`** — รายการ IP/hostname ของ server ที่จะ deploy
- **`registry`** — credential สำหรับ push/pull image — `password` อ่านจาก env var ชื่อ `KAMAL_REGISTRY_PASSWORD` (ค่ามาจาก `.env`)
- **`env.secret`** — รายการ env var ที่จะ inject เข้า container — Kamal อ่านจาก `.env` ของเครื่อง dev ตอน deploy
- **`ssh.user`** — user ที่ Kamal ssh เข้า server (default คือ `root`)
- **`volumes`** — bind-mount จาก host เข้า container `./hello-storage` คือ path relative ของ deploy user บน server (`/home/ubuntu/hello-storage`)

ในที่นี้เราจะใช้ registry ที่ [Docker Hub](https://hub.docker.com/)

จากนั้นไปดูไฟล์ `.env`:

```
KAMAL_REGISTRY_PASSWORD=change-this
RAILS_MASTER_KEY=another-env
```
{: file=".env"}

เราจะต้องสร้าง [access token](https://app.docker.com/settings/personal-access-tokens/create) เพื่อเอามากำหนดใน `KAMAL_REGISTRY_PASSWORD`

สำหรับ permissions จะให้เป็น Read & Write:

![](https://i.imgur.com/sYAt1YC.png)

เมื่อ Generate แล้วจะได้ access token มาแล้วก็ copy ไปใส่ `.env` ได้เลย:

![](https://i.imgur.com/ja50Ogd.png)

อีกค่าที่ต้องกำหนด `RAILS_MASTER_KEY` ให้ copy แล้วเอาไปใช้ได้ดังนี้:

```sh
cat config/master.key | pbcopy
```

ต้องไม่ลืมแก้ไข `database.yml` เสียก่อน — Rails default จะวาง SQLite ที่ `db/production.sqlite3` แต่เราต้องการให้อยู่ใน `storage/` ที่ bind-mount ออกมา ไม่งั้น database จะหายทุกครั้งที่ deploy ใหม่ (เพราะ container ใหม่ไม่มี database จาก container เก่า):

```yaml
production:
  <<: *default
  database: storage/production.sqlite3
```
{: file="config/database.yml"}

เสร็จแล้วก็ให้ `git add` และ `git commit` เสียก่อน

ต่อไปให้รัน Docker Desktop จากนั้นรันคำสั่งนี้:

```sh
kamal setup
```

`kamal setup` ทำทุกอย่างในคำสั่งเดียว: install Docker บน server (ถ้ายังไม่มี), login เข้า Docker Hub, build image บนเครื่อง dev, push ขึ้น registry, ssh เข้า server pull image ลงมา, start Traefik (reverse proxy), start app container, ผูก app เข้ากับ Traefik

รอจนเสร็จ แล้วตรวจดูผล:

```sh
open http://SERVER_ADDRESS/up
open http://SERVER_ADDRESS/posts
```

เสร็จ!!!

ครั้งต่อไปใช้ `kamal deploy` พอ — แค่ build + push + rolling update container

## Troubleshooting

**`unauthorized: incorrect username or password` ตอน push image**
ยังไม่ได้ generate Docker Hub access token หรือใส่ผิดใน `.env` — ตรวจค่า `KAMAL_REGISTRY_PASSWORD` ตรงกับ token ที่สร้างไว้

**`permission denied` ตอนเขียน SQLite ใน production**
`hello-storage` บน server permission ไม่พอ — ลอง `ls -la hello-storage` ถ้าไม่ใช่ `drwxrwxrwx` ให้ `chmod 777` (หรือ 770 + ตั้ง group ตามที่ note ไว้ข้างบน)

**`Browser blocks HTTP — redirect to HTTPS` ตอนเปิด `/up`**
ลืมตั้ง `config.force_ssl = false` หรือยังไม่ได้ commit แก้แล้ว `kamal deploy` ใหม่

## Conclusion

ยังต้องมีอะไรต้องทำอีกเยอะหลังจากนี้ ไม่ว่าจะเป็นการทำ SSL, เปลี่ยน database ไปใช้ตัวอื่น, ใช้ Nginx อะไรเหล่านี้ และที่ต้องหาเพิ่มคือจะใช้ registry ที่ไหนได้บ้างนอกจาก Docker Hub เพราะว่ามันได้แค่ 1 private repository เท่านั้นเอง

> ปัญหา "registry ฟรีได้แค่ 1 private repo" ได้รับการแก้ไขใน Kamal 2 ด้วย **Registry-Free Deployment** ที่รัน Docker registry บน deploy server เลย ดูวิธีใช้ที่ [Deploy Rails 8.1 with Kamal on a VM](/posts/deploy-rails-8-1-with-kamal-on-a-vm-experimental/)
{: .prompt-tip }

## References

- [Kamal documentation](https://kamal-deploy.org/)
- [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- <https://note.com/usutani/n/n890c38e68eb7>
