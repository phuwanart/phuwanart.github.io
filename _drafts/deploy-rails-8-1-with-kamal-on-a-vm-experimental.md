---
layout: post
title: Deploy Rails 8.1 with Kamal on a VM (Experimental)
---

## Prerequisites

```sh
sudo addgroup docker
sudo usermod -aG docker $USER
```

จากนั้น logout แล้ว login ใหม่ (เพื่อให้สิทธิ์กลุ่ม docker มีผล)

```sh
mkdir app_name_storage
```

## References

- https://alec-c4.com/posts/2025-04-02-kamal/