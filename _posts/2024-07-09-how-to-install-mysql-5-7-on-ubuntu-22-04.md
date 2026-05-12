---
layout: post
title: How to install MySQL 5.7 on Ubuntu 22.04
categories: Posts
tags: ubuntu mysql mysql 5.7 mysql5
date: 2024-07-09 11:40 +0700
---

บน Ubuntu ใหม่ ๆ จะสามารถลง MySQL ได้ ซึ่งจะเป็นเวอร์ชั่น 8 ซึ่งก็ดีอยู่แล้ว แต่ในบางสถานการณ์เรายังต้องใช้ MySQL 5 อยู่ — เช่น legacy app ที่ schema หรือ stored procedure ยังพึ่ง syntax/behavior ของ 5.7, vendor app ที่ support เฉพาะ 5.x, หรือต้องสำเนา production database ที่ยังเป็น 5.7 มาใช้บนเครื่อง dev — และไม่สามารถจะ `apt install` ได้โดยตรงแล้ว ทีนี้จะทำไงดี

> **MySQL 5.7 หมด support ตั้งแต่ตุลาคม 2023** Oracle ไม่ออก security update ให้แล้ว ใช้วิธีในโพสต์นี้เป็น **workaround ระยะสั้น** สำหรับ legacy app เท่านั้น แผนระยะยาวควรเป็น migrate ไป MySQL 8.0+ หรือ MariaDB
{: .prompt-warning }

## Prerequisites

- Ubuntu 22.04 LTS (ทดสอบใช้ได้กับ 24.04 ด้วย — มี callout เพิ่มเติมในขั้นตอน install)
- sudo access บน server

## ดาวน์โหลด mysql-apt-config

โชคดีที่ MySQL มี package repository เป็นของตัวเอง ซึ่งสามารถไปโหลดได้จากที่นี่ [MySQL APT repository download page](https://dev.mysql.com/downloads/repo/apt/) ซึ่งจากที่ทดลองเวอร์ชั่นล่าสุด ก็ไม่มี MySQL 5 เลือกติดตั้งได้แล้ว เพราะฉะนั้นเวอร์ชั่นล่าสุดที่มี MySQL 5 ให้เลือกลงได้นั้นคือ mysql-apt-config 0.8.15 ให้ทำการดาวน์โหลดตามนี้:

```sh
wget http://repo.mysql.com/mysql-apt-config_0.8.15-1_all.deb
```

จากนั้นก็ติดตั้งด้วยคำสั่ง:

```sh
sudo dpkg -i mysql-apt-config_0.8.15-1_all.deb
```

แล้วจะมีหน้าจอมาให้เลือกเวอร์ชั่นของ Ubuntu ให้เราเลือก `bionic` เพราะจากที่ลองเวอร์ชั่นนี้จะมี MySQL 5 ให้เลือก:

![](https://i.imgur.com/iAcf4V4.png)

จากนั้นเลือกเมนูแรก ซึ่งจะเห็นว่าเป็นเวอร์ชั่น 8 อยู่:

![](https://i.imgur.com/S2T4kYc.png)

เมื่อเข้ามาแล้วจะเห็น MySQL 5.7 ให้เลือก ก็เลือกเลย:

![](https://i.imgur.com/fzvtwTA.png)

จะเห็นว่าตอนนี้เราเลือกที่จะลงเวอร์ชั่น 5.7 แล้ว จากนั้นให้เลือก `Ok`:

![](https://i.imgur.com/hj4XDiA.png)

เมื่อเสร็จแล้วมันจะไปสร้าง apt source ให้ใน `/etc/apt/sources.list.d/`:

```sh
$ cat /etc/apt/sources.list.d/mysql.list
### THIS FILE IS AUTOMATICALLY CONFIGURED ###
# You may comment out entries below, but any other modifications may be lost.
# Use command 'dpkg-reconfigure mysql-apt-config' as root for modifications.
deb http://repo.mysql.com/apt/ubuntu/ bionic mysql-apt-config
deb http://repo.mysql.com/apt/ubuntu/ bionic mysql-5.7
deb http://repo.mysql.com/apt/ubuntu/ bionic mysql-tools
#deb http://repo.mysql.com/apt/ubuntu/ bionic mysql-tools-preview
deb-src http://repo.mysql.com/apt/ubuntu/ bionic mysql-5.7
```
{: file="/etc/apt/sources.list.d/mysql.list"}

## แก้ GPG signature error

จากนั้นให้เราทำการ `apt update` อาจจะเจอ error ดังนี้:

```sh
$ sudo apt update
Hit:1 http://th.archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://th.archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]
Get:3 http://repo.mysql.com/apt/ubuntu bionic InRelease [20.0 kB]
Get:4 https://download.docker.com/linux/ubuntu jammy InRelease [48.8 kB]
Hit:5 http://th.archive.ubuntu.com/ubuntu jammy-backports InRelease
Get:6 http://security.ubuntu.com/ubuntu jammy-security InRelease [129 kB]
Err:3 http://repo.mysql.com/apt/ubuntu bionic InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B7B3B788A8D3785C
...
W: GPG error: http://repo.mysql.com/apt/ubuntu bionic InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B7B3B788A8D3785C
E: The repository 'http://repo.mysql.com/apt/ubuntu bionic InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

![](https://i.imgur.com/aenH6ao.png)

ให้ดูที่บรรทัดที่มี `NO_PUBKEY`:

```
W: GPG error: ... NO_PUBKEY B7B3B788A8D3785C
```

เราจะเอา key นี้เพิ่มลงไปด้วยคำสั่งตามนี้ โดย key ในที่นี้คือ `B7B3B788A8D3785C`:

```sh
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys B7B3B788A8D3785C
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
Executing: /tmp/apt-key-gpghome.X9f7mhDRYw/gpg.1.sh --keyserver keyserver.ubuntu.com --recv-keys B7B3B788A8D3785C
gpg: key B7B3B788A8D3785C: public key "MySQL Release Engineering <mysql-build@oss.oracle.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

แล้วทำการ `apt update` อีกที:

```sh
$ sudo apt update
...
W: http://repo.mysql.com/apt/ubuntu/dists/bionic/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
```

![](https://i.imgur.com/md1fs8S.png)

## แก้ deprecation warning (legacy keyring)

ยังไม่จบ เราต้องแก้การแจ้งเตือนนี้เสียก่อน:

```
W: http://repo.mysql.com/apt/ubuntu/dists/bionic/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
```

เริ่มจากดู GPG keys ที่มีอยู่ในเครื่องก่อน:

```sh
$ sudo apt-key list
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
/etc/apt/trusted.gpg
--------------------
pub   dsa1024 2003-02-03 [SCA] [expired: 2022-02-16]
      A4A9 4068 76FC BD3C 4567  70C8 8C71 8D3B 5072 E1F5
uid           [ expired] MySQL Release Engineering <mysql-build@oss.oracle.com>

pub   rsa4096 2023-10-23 [SC] [expires: 2025-10-22]
      BCA4 3417 C3B4 85DD 128E  C6D4 B7B3 B788 A8D3 785C
uid           [ unknown] MySQL Release Engineering <mysql-build@oss.oracle.com>
sub   rsa4096 2023-10-23 [E] [expires: 2025-10-22]
...
```

ในส่วนนี้เราจะเจอ key ของ MySQL Release Engineering อยู่ 2 ตัว เราจะไม่สนใจตัวที่ expired ให้เราดูที่ unknown:

```
pub   rsa4096 2023-10-23 [SC] [expires: 2025-10-22]
      BCA4 3417 C3B4 85DD 128E  C6D4 B7B3 B788 A8D3 785C
uid           [ unknown] MySQL Release Engineering <mysql-build@oss.oracle.com>
```

เราจะเห็นชุดของ key ดังนี้ `BCA4 3417 C3B4 85DD 128E  C6D4 B7B3 B788 A8D3 785C` เราจะใช้แค่ 8 ตัวสุดท้ายคือ `A8D3 785C` (ไม่เอาช่องว่าง) ไปเพิ่มลงใน GPG key โดยสร้างเป็นไฟล์อยู่ภายใต้ `/etc/apt/trusted.gpg.d` directory:

```sh
$ sudo apt-key export A8D3785C | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/mysql.gpg
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
```

เสร็จแล้วก็ `apt update` ก็พบว่าแจ้งเตือนต่าง ๆ หายไปแล้ว:

```sh
$ sudo apt update
Hit:1 http://th.archive.ubuntu.com/ubuntu jammy InRelease
Hit:2 https://download.docker.com/linux/ubuntu jammy InRelease
Hit:3 http://security.ubuntu.com/ubuntu jammy-security InRelease
Hit:4 http://th.archive.ubuntu.com/ubuntu jammy-updates InRelease
Hit:5 http://th.archive.ubuntu.com/ubuntu jammy-backports InRelease
Hit:6 http://repo.mysql.com/apt/ubuntu bionic InRelease
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
```

## ตั้ง apt preferences ให้เลือก MySQL repo

ต่อไปเราจะ install MySQL กันแล้ว แต่ว่า Ubuntu ก็ยังเลือกเวอร์ชั่น 8 ให้อยู่:

```sh
$ sudo apt install mysql-server
...
The following NEW packages will be installed:
  ... mysql-client-8.0 mysql-client-core-8.0 mysql-common mysql-server mysql-server-8.0
  mysql-server-core-8.0
...
Do you want to continue? [Y/n] n
Abort.
```

ให้เรายกเลิกการติดตั้งไปก่อน

เราจะต้องให้ apt ไปเรียก package จาก MySQL repository โดยเราต้องกำหนดลำดับก่อน โดยจะกำหนดใน `/etc/apt/preferences.d/mysql` ดังนี้:

```
Package: *
Pin: origin repo.mysql.com
Pin-Priority: 900

Package: *
Pin: origin ch.archive.ubuntu.com
Pin-Priority: 700

Package: *
Pin: origin security.ubuntu.com
Pin-Priority: 800
```
{: file="/etc/apt/preferences.d/mysql"}

`Pin-Priority` ที่สูงกว่า (900) บอก apt ว่าถ้า package มีในหลาย source ให้เลือกจาก `repo.mysql.com` ก่อน — ทำให้ install ได้ MySQL 5.7 แทน MySQL 8 จาก default Ubuntu repo

## ติดตั้ง mysql-server

จากนั้นก็ install อีกครั้ง:

```sh
$ sudo apt install mysql-server
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libmecab2 libtinfo5 mysql-client mysql-common mysql-community-client mysql-community-server
The following NEW packages will be installed:
  libmecab2 libtinfo5 mysql-client mysql-common mysql-community-client mysql-community-server mysql-server
0 upgraded, 7 newly installed, 0 to remove and 7 not upgraded.
Need to get 55.6 MB of archives.
After this operation, 327 MB of additional disk space will be used.
Do you want to continue? [Y/n] Y
```

> อัพเดทสำหรับ **Ubuntu 24.04** พอมาถึงขั้นตอนนี้จะเจอ error ดังนี้:
>
> ```sh
> $ sudo apt install mysql-server
> ...
> The following packages have unmet dependencies:
>  mysql-community-server : Depends: mysql-client (= 5.7.42-1ubuntu18.04) but it is not going to be installed
>                           Depends: libaio1 (>= 0.3.93) but it is not installable
> E: Unable to correct problems, you have held broken packages.
> ```
>
> เพราะว่าต้องการลง `libaio1` ก่อนโดยต้องลงเองดังนี้:
>
> ```sh
> curl -O http://launchpadlibrarian.net/646633572/libaio1_0.3.113-4_amd64.deb
> sudo dpkg -i libaio1_0.3.113-4_amd64.deb
> ```
>
> จากนั้นก็จะเจอ error ที่ต้องติดตั้ง lib อีกตัวนึง:
>
> ```sh
> $ sudo apt install mysql-server
> ...
> The following packages have unmet dependencies:
>  mysql-community-client : Depends: libtinfo5 (>= 6) but it is not installable
> E: Unable to correct problems, you have held broken packages.
> ```
>
> ให้ติดตั้ง lib นี้ตามนี้:
>
> ```sh
> curl -O http://launchpadlibrarian.net/648013231/libtinfo5_6.4-2_amd64.deb
> sudo dpkg -i libtinfo5_6.4-2_amd64.deb
> ```
>
> พอติดตั้ง lib เสร็จก็น่าจะสามารถลง MySQL 5.7 ผ่านแล้ว
{: .prompt-danger }

ทำตามขั้นตอนก็จะเห็นว่า MySQL ที่ระบบลงให้เป็นเวอร์ชั่น 5 แล้ว:

```sh
$ mysql --version
mysql  Ver 14.14 Distrib 5.7.42, for Linux (x86_64) using  EditLine wrapper
```

ตรวจสอบ MySQL package ที่ลงไป:

```sh
$ dpkg -l | grep mysql
ii  mysql-apt-config                       0.8.15-1                                all          Auto configuration for MySQL APT Repo.
ii  mysql-client                           5.7.42-1ubuntu18.04                     amd64        MySQL Client meta package depending on latest version
ii  mysql-common                           5.7.42-1ubuntu18.04                     amd64        MySQL Common
ii  mysql-community-client                 5.7.42-1ubuntu18.04                     amd64        MySQL Client
ii  mysql-community-server                 5.7.42-1ubuntu18.04                     amd64        MySQL Server
ii  mysql-server                           5.7.42-1ubuntu18.04                     amd64        MySQL Server meta package depending on latest version
```

และตรวจดู status ของ MySQL:

```sh
$ systemctl status mysql
● mysql.service - MySQL Community Server
     Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-07-09 04:19:07 UTC; 2min 54s ago
    ...
```

## Post-install: `mysql_secure_installation`

หลัง install MySQL เพิ่งติดตั้งใหม่จะมีค่า default ที่ไม่ปลอดภัย (root password ว่าง, anonymous user, test database) รัน script ที่มาให้:

```sh
sudo mysql_secure_installation
```

จะถามคำถามต่อเนื่อง — ตอบตามนี้:

- **Set root password** — เซ็ตเลย ตั้งให้แข็งแรง
- **Remove anonymous users** — Yes
- **Disallow root login remotely** — Yes (ถ้าจะ access จากเครื่องอื่น สร้าง user แยกแทน)
- **Remove test database** — Yes
- **Reload privilege tables** — Yes

## Create database สำหรับ Rails app

login เข้า MySQL ด้วย root:

```sh
sudo mysql -u root -p
```

สร้าง database และ user สำหรับ Rails app — ไม่ใช้ root ใน production:

```sql
CREATE DATABASE my_app_production CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'my_app'@'localhost' IDENTIFIED BY 'strong_password_here';
GRANT ALL PRIVILEGES ON my_app_production.* TO 'my_app'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

ตั้งใน `config/database.yml` ของ Rails:

```yaml
production:
  <<: *default
  database: my_app_production
  username: my_app
  password: <%= ENV["MY_APP_DATABASE_PASSWORD"] %>
  host: localhost
```
{: file="config/database.yml"}

`utf8mb4` charset รองรับ emoji และ Unicode supplementary plane ใช้ตอน collate `utf8mb4_unicode_ci` ก็พอ (MySQL 8+ default เปลี่ยนเป็น `utf8mb4_0900_ai_ci` แต่ 5.7 ยังไม่มี)

## Hold mysql-apt-config

อย่างสุดท้าย เราต้องระวังการ upgrade จาก Ubuntu ซึ่งอาจจะ upgrade MySQL 5 ไปเป็น 8 ให้เราเองโดยที่เราไม่ตั้งใจ:

```sh
$ sudo apt upgrade
...
The following packages will be upgraded:
  mysql-apt-config
1 upgraded, 0 newly installed, 0 to remove and 6 not upgraded.
...
Do you want to continue? [Y/n] n
Abort.
```

จะเห็นว่ามีการจะอัพเกรด `mysql-apt-config` เราจะ hold การ upgrade ไปก่อนด้วย:

```sh
sudo apt-mark hold mysql-apt-config
```

เช็คว่า hold สำเร็จ:

```sh
sudo apt-mark showhold
```

## ทางเลือก: Docker

ถ้าไม่อยาก pollute apt source ของ host และอยากให้ isolate ง่าย ใช้ container `mysql:5.7` แทนคือทางเลือกที่คลีนกว่า — ไม่ต้องตั้ง preferences, ไม่ต้องห่วงเรื่อง upgrade accidentally, แค่ลบ volume ก็เริ่มใหม่ได้:

```yaml
services:
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: my_app_development
      MYSQL_USER: my_app
      MYSQL_PASSWORD: app_password
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"

volumes:
  mysql_data:
```
{: file="docker-compose.yml"}

```sh
docker compose up -d
```

แต่ถ้า production บน server เดียวกับ Rails และไม่อยากเพิ่ม Docker เป็น dependency การลงระบบ native ตามวิธีในโพสต์ก็เป็นทางเลือกที่สมเหตุสมผล

## Conclusion

ติดตั้ง MySQL 5.7 บน Ubuntu 22.04/24.04 ผ่านขั้นตอน MySQL APT repo + apt preferences เรียบร้อย — แต่อย่าลืมว่า MySQL 5.7 EOL ไปแล้ว ใช้เป็น workaround ระยะสั้นเท่านั้น เมื่อมีโอกาสควรวางแผน migrate ขึ้น MySQL 8 หรือ MariaDB

## References

- [Install MySQL 5.7 on Ubuntu 20.04 — Claudio Kuenzler](https://www.claudiokuenzler.com/blog/991/install-mysql-5.7-on-ubuntu-20.04-focal-avoid-8.0-packages)
- [How to install MySQL on Ubuntu — Devart](https://www.devart.com/dbforge/mysql/how-to-install-mysql-on-ubuntu/)
- [Fix `NO_PUBKEY` GPG error — Chris Jean](https://chrisjean.com/fix-apt-get-update-the-following-signatures-couldnt-be-verified-because-the-public-key-is-not-available/)
- [Key is stored in legacy trusted.gpg keyring — It's FOSS](https://itsfoss.com/key-is-stored-in-legacy-trusted-gpg/)
- [Installation failed in Ubuntu 24.04 LTS — Local Community](https://community.localwp.com/t/installation-failed-in-ubuntu-24-04-lts/42579/3)
- [MySQL 5.7 End of Life announcement](https://www.oracle.com/us/support/library/lifetime-support-technology-069183.pdf)
