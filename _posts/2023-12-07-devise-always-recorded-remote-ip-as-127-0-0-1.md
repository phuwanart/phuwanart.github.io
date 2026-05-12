---
layout: post
title: Devise always recorded remote_ip as 127.0.0.1
categories: Posts
tags: rails nginx
date: 2023-12-07 13:34 +0700
---

## Problem

ใน Devise หลังจากเพิ่ม module `:trackable` เข้าไปแล้ว ปรากฏว่า IP ที่ track ได้มักจะเป็น `127.0.0.1` ตลอด

## Root cause

เมื่อ Nginx proxy request ไปที่ Puma/Unicorn ผ่าน localhost (unix socket หรือ TCP) ฝั่ง Rails จะเห็น `remote_addr` เป็น `127.0.0.1` เพราะ Nginx คือ "client" ของ Rails — IP จริงของผู้ใช้ต้องถูกส่งมาผ่าน header

Rails มี middleware ชื่อ `ActionDispatch::RemoteIp` ที่หา client IP จากหลาย header ตามลำดับ:

1. `Client-Ip` (จาก env `HTTP_CLIENT_IP`)
2. `X-Forwarded-For` (ลบ trusted proxy ออกแล้ว)
3. fall back เป็น `REMOTE_ADDR`

ปัญหาคือ `X-Forwarded-For` ที่ Nginx ตั้งให้ผ่าน `$proxy_add_x_forwarded_for` จะ append IP ของตัวเอง (`127.0.0.1`) เข้าไป — ซึ่ง Rails ถือว่าเป็น trusted proxy และกรองออก ทำให้สุดท้ายเหลือว่างหรือ fall back มาที่ `127.0.0.1`

## Solution

เพิ่ม `proxy_set_header HTTP_CLIENT_IP $remote_addr;` ใน `/etc/nginx/nginx.conf`:

```nginx
location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header HTTP_CLIENT_IP $remote_addr;  # <-- add this line
    proxy_redirect off;
    proxy_pass http://unicorn;
}
```
{: file="/etc/nginx/nginx.conf"}

เสร็จแล้ว restart nginx การส่ง header `HTTP_CLIENT_IP` ตรงๆ ข้าม logic การกรอง trusted proxy ที่กล่าวข้างบน Rails หยิบค่าจาก `Client-Ip` ก่อน `X-Forwarded-For` เลย

### ทางเลือก: ตั้ง `trusted_proxies` ใน Rails

แทนการแก้ที่ Nginx แก้ที่ Rails ก็ได้ — บอก Rails ว่า IP ไหนที่ตั้งใจเป็น proxy (ให้กรองออก) แล้วที่เหลือใน `X-Forwarded-For` คือ IP จริงของ client:

```ruby
config.action_dispatch.trusted_proxies = [
  IPAddr.new("127.0.0.1"),
  IPAddr.new("::1")
]
```
{: file="config/application.rb"}

ทางนี้คลีนกว่าเพราะใช้ standard `X-Forwarded-For` ที่ Nginx ตั้งให้อยู่แล้ว ไม่ต้องเพิ่ม custom header — แต่ workaround ใน solution หลักก็ใช้ได้ดี เลือกตามสะดวก

## Conclusion

เป็นปัญหาที่การ config nginx ไม่เกี่ยวอะไรกับ Devise — Devise แค่อ่านค่า `request.remote_ip` มาเก็บ ถ้า Rails ได้ค่าถูก Devise ก็เก็บค่าถูก

## References

- [Devise issue #1186](https://github.com/heartcombo/devise/issues/1186)
- [Rails Guides — Action Dispatch and Trusted Proxies](https://guides.rubyonrails.org/configuring.html#config-action-dispatch-trusted-proxies)
- [`ActionDispatch::RemoteIp` source](https://github.com/rails/rails/blob/main/actionpack/lib/action_dispatch/middleware/remote_ip.rb)
