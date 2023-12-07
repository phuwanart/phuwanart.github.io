---
layout: post
title: Devise always recorded remote_ip as 127.0.0.1
categories: Posts
tags: rails nginx
date: 2023-12-07 13:34 +0700
---
## Problem

ใน Devise หลังจากเพิ่ม modules `:trackable` เข้าไปแล้ว ปรากฏว่า IP ที่ track ได้มักจะเป็น `127.0.0.1` ตลอด

## Solution

เพิ่ม `proxy_set_header HTTP_CLIENT_IP $remote_addr;` ใน `/etc/nginx/nginx.conf`

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
{:file='/etc/nginx/nginx.conf'}

เสร็จแล้ว restart nginx

## References

- [https://github.com/heartcombo/devise/issues/1186](https://github.com/heartcombo/devise/issues/1186)