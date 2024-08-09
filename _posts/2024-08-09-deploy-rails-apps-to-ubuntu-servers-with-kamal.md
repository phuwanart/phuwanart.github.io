---
layout: post
title: Deploy Rails apps to Ubuntu servers with Kamal
categories: Posts
tags: deploy rails ubuntu kamal docker
date: 2024-08-09 15:10 +0700
---
[Kamal](https://kamal-deploy.org/) เป็นเครื่องมือที่ใช้สำหรับ deploy web app ด้วย Docker ในที่นี้เราจะลองใช้เครื่องมือนี้ deploy ลอง server ของเราเอง

## Preparing Server

เริ่มจากที่เราติดตั้ง ubuntu server หลังจากติดตั้งเสร็จให้ทำการตั้งค่า firewall[^config-firewall] ให้เรียบร้อยเสียก่อน

[^config-firewall]: [Config firewall](https://phuwanart.github.io/posts/how-to-deploy-rails-7-2-from-scratch-with-capistrano-puma-and-nginx-on-ubuntu-24-04/#setup-firewall)

จากนั้นตรวจดูว่าเราสามารถ ssh เข้าไปได้:

```sh
ssh ubuntu@SERVER_ADDRESS
exit
```

แล้วก็ทำการอนุญาตให้ login ได้โดยไม่ต้องใช้ password:

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

- ตั้งค่า Docker's `apt` repository:

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

ใน rails application ที่เราจะสร้าง เราจะใช้ SQLite ในการทดลองนี้ เพราะฉะนั้นเราจะต้องสร้าง directory เพื่อให้ docker มา mounted ไฟล์ไว้ที่นี่:

```sh
mkdir hello-storage
sudo chmod 777 hello-storage
exit
```

## Preparing Rails

### Prerequisites

สำหรับเครื่อง local ที่ต้องเตรียมสำหรับสร้าง rails application มีดังต่อไปนี้:

- Ruby 3.3.4
- Rails 7.2.0.rc1 
- Kamal 1.8.1
- Docker Desktop

### Create new rails application

```sh
rails new hello
cd hello
bundle lock --add-platform aarch64-linux
bin/rails g scaffold Post body:text
bin/setup
```

แก้ไข routes.rb:

```ruby
Rails.application.routes.draw do
  resources :posts
  root "posts#index"
end
```
{:file='config/routes.rb'}

แก้ไขยังไม่ force ssl ก่อน:

```ruby
config.force_ssl = false
```
{:file='config/environments/production.rb'}

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
{:file='config/deploy.yml'}


ในที่นี้เราจะใช้ registry ที่ [docker hub](https://hub.docker.com/)

จากนั้นไปดูไฟล์ `.env`:

```
KAMAL_REGISTRY_PASSWORD=change-this
RAILS_MASTER_KEY=another-env
```
{:file='.env'}

เราจะต้องสร้าง [access token](https://app.docker.com/settings/personal-access-tokens/create) เพื่อเอามากำหนดใน `KAMAL_REGISTRY_PASSWORD`

สำหรับ permissions จะให้เป็น Read & Write:

![](https://i.imgur.com/sYAt1YC.png)

เมื่อ Generate แล้วจะได้ access token มาแล้วก็ copy ไปใส่ `.env` ได้เลย:

![](https://i.imgur.com/ja50Ogd.png)

อีกค่าที่ต้องกำหนด `RAILS_MASTER_KEY` ให้ copy แล้วเอาไปใช้ได้ดังนี้:

```sh
cat config/master.key | pbcopy
```

ต้องไม่ลืมแก้ไข `database.yml` เสียก่อน:

```yaml
production:
  <<: *default
  database: storage/production.sqlite3
```
{:file='config/database.yml'}

เสร็จแล้วก็ให้ `git add` และ `git commit` เสียก่อน

ต่อไปให้รัน Docker Desktop จากนั้นรันคำสั่งนี้:

```sh
kamal setup
```
รอจนเสร็จ แล้วตรวจดูผล:

```sh
open http://SERVER_ADDRESS/up
open http://SERVER_ADDRESS/posts
```

เสร็จ!!!

## Conclusion

ยังต้องมีอะไรต้องทำอีกเยอะหลังจากนี้ ไม่ว่าจะเป็นการทำ ssl, เปลี่ยน database ไปใช้ตัวอื่น, ใช้ nginx อะไรเหล่านี้ และที่ต้องหาเพิ่มคือจะใช้ registry ที่ไหนได้บ้างนอกจาก docker hub เพราะว่ามันได้แค่ 1 private repository เท่านั้นเอง

## Reference

- https://note.com/usutani/n/n890c38e68eb7
