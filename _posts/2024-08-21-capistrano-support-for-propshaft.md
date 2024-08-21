---
layout: post
title: Capistrano support for propshaft
categories: Posts
tags: deploy rails propshaft capistrano
date: 2024-08-21 21:45 +0700
---
[Propshaft](https://github.com/rails/propshaft) เป็น asset pipeline ใหม่ของ rails ในกรณีนี้หากเราใช้ propshaft และ deploy ด้วย capistrano จะ deploy ไม่ผ่าน เราจะต้องเพิ่มโค้ดนี้ลงใน `config/deploy.rb` เสียก่อน:

```ruby
set :assets_manifests, -> {
    [release_path.join("public", fetch(:assets_prefix), '.manifest.json')]
}
```
{:file='config/deploy.rb'}

## References

- https://github.com/capistrano/rails/issues/257