---
layout: post
title: Capistrano support for propshaft
categories: Posts
tags: deploy rails propshaft capistrano
date: 2024-08-21 21:45 +0700
---

[Propshaft](https://github.com/rails/propshaft) เป็น asset pipeline ใหม่ของ Rails (default ของ Rails 8) ที่เบากว่า Sprockets เพราะไม่ทำ transpile/concatenate ปล่อยให้ bundler ภายนอก (importmap, jsbundling, cssbundling) จัดการแทน

ปัญหาคือถ้า deploy ด้วย Capistrano แล้ว app ใช้ Propshaft จะเจอ error ลักษณะนี้ตอน deploy:

```
No such file or directory @ rb_check_realpath_internal —
.../current/public/assets/.sprockets-manifest-XXX.json
```

เพราะ task `capistrano/rails/assets` ใช้ `assets_manifests` ค้นหา manifest file ของแต่ละ release เพื่อทำ asset bridging (กัน in-flight request 404 ขณะ deploy ใหม่ โดยเก็บ asset ของ release เก่าไว้ในระหว่าง symlink switch) — แต่ default ยังชี้ไปที่ Sprockets manifest (`.sprockets-manifest-*.json`) ส่วน Propshaft output เป็น `.manifest.json` เฉยๆ

แก้ด้วยการ override `assets_manifests` ใน `config/deploy.rb` หลังบรรทัด `require "capistrano/rails/assets"`:

```ruby
set :assets_manifests, -> {
  [release_path.join("public", fetch(:assets_prefix), ".manifest.json")]
}
```
{: file="config/deploy.rb"}

แค่นี้ deploy ก็ผ่านเหมือนเดิม

## References

- [Propshaft README](https://github.com/rails/propshaft)
- [capistrano-rails README](https://github.com/capistrano/rails)
- <https://github.com/capistrano/rails/issues/257>
