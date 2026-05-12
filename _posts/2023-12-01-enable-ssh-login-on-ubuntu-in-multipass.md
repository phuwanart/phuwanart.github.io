---
layout: post
title: Enable SSH login on Ubuntu in Multipass
date: 2023-12-01 23:25 +0700
categories: Posts
tags: multipass ssh ubuntu
---

[Multipass](https://multipass.run) เป็น VM tool ของ Canonical ที่รัน Ubuntu บน Mac/Windows/Linux ได้ง่ายๆ เหมาะกับการ test deployment script ในเครื่องท้องถิ่นโดยไม่ต้องเช่า VPS

ปกติเรา login เข้า VM ด้วย:

```sh
$ multipass shell primary
```
{: file="Local Machine"}

แต่ว่ามีงานบางงานที่เราต้องใช้การ login เข้า VM ด้วย SSH อย่างเช่น จะทดสอบการ deploy โดยใช้พวก deployment automation tool ต่าง ๆ (Capistrano, Ansible, Kamal) ที่เชื่อมผ่าน SSH เท่านั้น

ที่นี้ลอง SSH เข้า VM ดู โดยเราสามารถดู IP ได้ด้วย:

```sh
$ multipass list
Name                    State             IPv4             Image
primary                 Running           192.168.64.32    Ubuntu 22.04 LTS
```
{: file="Local Machine"}

จากนั้นลอง SSH จะพบว่าไม่สามารถเข้าได้:

```sh
$ ssh ubuntu@192.168.64.32
The authenticity of host '192.168.64.32 (192.168.64.32)' can't be established.
...
ubuntu@192.168.64.32: Permission denied (publickey).
```
{: file="Local Machine"}

เพราะ Multipass default authorize เฉพาะ key ของตัวเองใน `authorized_keys` (ที่ใช้กับ `multipass shell`) — ไม่ใช่ key ของ user

ต่อไปเราจะ copy SSH key จากเครื่องเราโดยดู key ได้ด้วย:

```sh
$ cat ~/.ssh/id_ed25519.pub
ssh-ed25519 ...
```
{: file="Local Machine"}

> ถ้ายังไม่มี key ให้สร้างด้วย `ssh-keygen -t ed25519` ก่อน — แนะนำ ed25519 เพราะเร็วกว่า ปลอดภัยกว่า และ key สั้นกว่า RSA ถ้า server เก่ามากที่รองรับเฉพาะ RSA ใช้ `cat ~/.ssh/id_rsa.pub` แทน
{: .prompt-tip }

copy ส่วน `ssh-ed25519 ...` (หรือ `ssh-rsa ...`) จากนั้นเราจะ shell เข้า VM ตัวที่เราต้องการ:

```sh
$ multipass shell primary
```
{: file="Local Machine"}

แล้วก็เพิ่ม SSH key ใน `~/.ssh/authorized_keys` ด้วย:

```sh
ubuntu@primary:~$ echo 'ssh-ed25519 ...' >> ~/.ssh/authorized_keys
```
{: file="ubuntu@primary"}

ออกจาก VM ไปที่ local ของเรา แล้วลอง SSH ดูอีกครั้ง แต่ถ้ายังเข้าไม่ได้ให้ใช้คำสั่งอีก 2 คำสั่ง:

```sh
$ eval $(ssh-agent)
$ ssh-add
```
{: file="Local Machine"}

`ssh-agent` คือ daemon ที่เก็บ private key ที่ unlock แล้วไว้ใน memory ส่วน `ssh-add` โหลด key เข้า agent บางครั้ง shell session ใหม่ยังไม่ได้ start agent หรือ key ยังไม่ได้ถูก load — ssh client เลยหา key ไม่เจอและ fall back ไป prompt password (แต่ Multipass VM ไม่ได้ตั้ง password ให้) ผลคือ permission denied

เรียบร้อย

## ทำให้ง่ายขึ้นด้วย `~/.ssh/config`

IP ของ Multipass VM เปลี่ยนตอน restart ทุกครั้ง การพิมพ์ IP ใหม่ทุกครั้งน่าเบื่อ และจะเจอ warning ว่า host key เปลี่ยนถ้า recreate VM — แก้ด้วยการตั้ง alias ใน `~/.ssh/config`:

```
Host multipass-primary
  HostName primary.local
  User ubuntu
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```
{: file="~/.ssh/config"}

ต่อไปนี้แค่ `ssh multipass-primary` ก็เข้าได้แล้ว — `primary.local` ใช้ mDNS ที่ Multipass register ให้ ทำให้ไม่ต้องสน IP, ส่วน `StrictHostKeyChecking no` กับ `UserKnownHostsFile /dev/null` กัน warning ตอน host key เปลี่ยนหลัง recreate VM (เหมาะกับ VM ทดสอบเท่านั้น — ไม่ควรใช้กับ production server)

## Troubleshooting

**`Permission denied (publickey)` แม้ copy key แล้ว**
ตรวจ permission ของไฟล์ใน VM:

```sh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
{: file="ubuntu@primary"}

sshd จะปฏิเสธ key ถ้า permission กว้างเกินไป (กลัวคนอื่นแก้ไขได้)

**`Could not resolve hostname primary.local`**
ระบบไม่ได้เปิด mDNS resolver บน macOS มีให้อยู่แล้ว (Bonjour) ส่วน Linux อาจต้องลง `avahi-daemon` หรือใช้ IP ตรงๆ แทน

## References

- [Multipass documentation](https://multipass.run/docs)
- [OpenSSH config manual](https://man.openbsd.org/ssh_config)
