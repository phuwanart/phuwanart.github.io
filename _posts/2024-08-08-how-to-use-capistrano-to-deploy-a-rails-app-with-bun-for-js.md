---
layout: post
title: How to use capistrano to deploy a Rails app with bun for js?
categories: Posts
tags: deploy rails ubuntu capistrano bun
date: 2024-08-08 14:17 +0700
---
## Problem

ตามหัวข้อ เราติดตั้ง [bun.sh](https://bun.sh/) บน Ubuntu ได้โดย:

```bash
curl -fsSL https://bun.sh/install | bash 
```

ซึ่งก็ติดตั้งได้โดยไม่มีปัญหาอะไร

แต่พอ deploy ด้วย capistrano กลับฟ้องข้อผิดพลาดนี้:

```bash
cssbundling-rails: Command install failed, ensure bun is installed
```

## Solution

เป็นเพราะ `.bashrc` ไม่รันเมื่อ capistrano login เข้ามา

ให้เพิ่มโค้ดนี้ลงใน `config/deploy.rb`:

```ruby
set :default_env, { path: "/home/[deploy_user]/.bun/bin:$PATH" }
```

## References

- https://capistranorb.com/documentation/getting-started/configuration/
- https://stackoverflow.com/questions/77533014/how-to-use-capistrano-to-deploy-a-rails-app-with-bun-for-js
