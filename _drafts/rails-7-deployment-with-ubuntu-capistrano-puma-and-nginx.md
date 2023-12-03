---
layout: post
title: Rails 7 Deployment with Ubuntu, Capistrano, Puma and Nginx
categories: Posts
tags: rails deploy ubuntu capistrano
---

ในที่นี้เราจะทำการ deploy rails ไปที่ ubuntu server กัน:

- Ubuntu 22.04 LTS
- Ruby 3.2.2
- Rails 7.0.8
- Puma 5

## Setup Ubuntu

หลังจากติดตั้ง ubuntu server ใหม่แล้วก็ทำการอัพเดทและรีบูทก่อน:

```sh
sudo apt update && sudo apt upgrade -y && sudo apt-get autoremove && sudo reboot
```

จากนั้นติดตั้ง `net-tools` เพื่อดู IP ของ server:

```sh
sudo apt install net-tools
```

และ `vim` เป็น text editor:

```sh
sudo apt install vim
```

### Install system dependencies

```sh
sudo apt install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates
```

### Connect as Root

```sh
ssh root@1.2.3.4
```
{:file='Local Machine'}
