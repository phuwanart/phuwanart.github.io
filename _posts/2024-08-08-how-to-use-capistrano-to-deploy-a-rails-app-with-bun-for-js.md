---
layout: post
title: How to use capistrano to deploy a Rails app with bun for js?
categories: Posts
tags: deploy rails ubuntu capistrano bun
date: 2024-08-08 14:17 +0700
---

## Problem

ตามหัวข้อ เราติดตั้ง [bun.sh](https://bun.sh/) บน Ubuntu ได้โดย:

```sh
curl -fsSL https://bun.sh/install | bash
```

ซึ่งก็ติดตั้งได้โดยไม่มีปัญหาอะไร

แต่พอ deploy ด้วย Capistrano กลับฟ้องข้อผิดพลาดนี้:

```
cssbundling-rails: Command install failed, ensure bun is installed
```

## Root cause

bun installer เพิ่ม `export PATH="$HOME/.bun/bin:$PATH"` ลงใน `.bashrc` ของ user ตอนติดตั้ง — แต่ `.bashrc` รันเฉพาะ **interactive shell** เท่านั้น Capistrano ssh เข้า server แบบ **non-interactive** เพื่อรันคำสั่ง init script เหล่านี้จึงไม่ถูกโหลด PATH ก็เลยไม่มี `~/.bun/bin` ทำให้ `bun` ไม่เจอ

## Solution

ให้เพิ่มโค้ดนี้ลงใน `config/deploy.rb`:

```ruby
set :default_env, { path: "/home/[deploy_user]/.bun/bin:$PATH" }
```
{: file="config/deploy.rb"}

`default_env` คือ option ของ Capistrano ที่ set environment variable ก่อนรันทุกคำสั่งบน remote — ใช้ pattern เดียวกันได้กับ tool อื่นที่ติดตั้งผ่าน curl|bash แล้วใส่ PATH ลง `.bashrc` เช่น rbenv, nvm, mise:

```ruby
set :default_env, {
  path: "/home/[deploy_user]/.bun/bin:/home/[deploy_user]/.rbenv/shims:$PATH"
}
```
{: file="config/deploy.rb"}

### ทางเลือก: system-wide PATH

แทนที่จะ hardcode path ใน Capistrano config อีกวิธีคือเพิ่ม PATH ในระดับ system ผ่าน `/etc/profile.d/`:

```sh
echo 'export PATH="/home/deploy/.bun/bin:$PATH"' | sudo tee /etc/profile.d/bun.sh
sudo chmod +x /etc/profile.d/bun.sh
```

ไฟล์ใน `/etc/profile.d/*.sh` ถูก source โดย shell login ทุก user รวม non-interactive session ของ Capistrano — คลีนกว่าเพราะไม่ต้องเก็บ deploy user path ใน repo และเปลี่ยน deploy user ภายหลังไม่ต้องแก้ `deploy.rb`

## References

- [Capistrano configuration — `default_env`](https://capistranorb.com/documentation/getting-started/configuration/)
- [Bun installation](https://bun.sh/docs/installation)
- <https://stackoverflow.com/questions/77533014/how-to-use-capistrano-to-deploy-a-rails-app-with-bun-for-js>
