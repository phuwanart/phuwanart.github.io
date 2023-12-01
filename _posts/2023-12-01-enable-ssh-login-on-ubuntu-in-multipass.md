---
layout: post
title: Enable SSH login on Ubuntu in Multipass
date: 2023-12-01 23:25 +0700
categories: Posts
tags: multipass
---

ปกติแล้วหากเราใช้ [Multipass](https://multipass.run) เป็น VM เราสามารถ login เข้า VM โดยใช้:

```sh
$ multipass shell primary
```

แต่ว่ามีงานบางงานที่เราต้องใช้การ login เข้า VM ด้วย ssh อย่างเช่น จะทดสอบการ deploy โดยใช้พวก deployment automation tools ต่าง ๆ

ที่นี้ลอง ssh เข้า VM ดู โดยเราสามารถดู IP ได้ด้วย:

```sh
$ multipass list
Name                    State             IPv4             Image
primary                 Running           192.168.64.32    Ubuntu 22.04 LTS
```

จากนั้นลอง ssh จะพบว่าไม่สามารถเข้าได้:

```sh
$ ssh ubuntu@192.168.64.32
The authenticity of host '192.168.64.32 (192.168.64.32)' can't be established.
...
ubuntu@192.168.64.32: Permission denied (publickey).
```

ต่อไปเราจะ copy SSH key จากเครื่องเราโดยดู key ได้ด้วย:

```sh
$ cat ~/.ssh/id_rsa.pub
ssh-rsa ...
```

copy ส่วน `ssh-rsa ...` จากนั้นเราจะ shell เข้า VM ตัวที่เราต้องการ:

```sh
$ multipass shell primary
```

แล้วก็เพิ่ม SSH Key ใน `~/.ssh/authorized_keys` ด้วย:

```sh
ubuntu@primary:~$ echo 'ssh-rsa ...' >> ~/.ssh/authorized_keys
```

ออกจาก VM ไปที่ local ของเรา แล้วลอง ssh ดูอีกครั้ง แต่ถ้ายังเข้าไม่ได้ให้ใช้คำสั่งอีก 2 คำสั่ง:

```bash
$ eval $(ssh-agent)  
$ ssh-add
```

เรียบร้อย