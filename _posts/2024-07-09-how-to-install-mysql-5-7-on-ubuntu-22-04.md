---
layout: post
title: How to install MySQL 5.7 on Ubuntu 22.04
categories: Posts
tags: ubuntu mysql5
date: 2024-07-09 11:40 +0700
---

บน ubuntu ใหม่ ๆ จะสามารถลง mysql ได้ ซึ่งจะเป็นเวอร์ชั่น 8 ซึ่งก็ดีอยู่แล้ว แต่ในบางสถานการณ์เรายังต้องใช้ mysql 5 อยู่ และไม่สามารถจะ `apt install` ได้โดยตรงแล้ว ทีนี้จะทำไงดี

โชคดีที่ mysql มี package repositories เป็นของตัวเอง ซึ่งสามารถไปโหลดได้จากที่นี่ [MySQL APT repository download page](https://dev.mysql.com/downloads/repo/apt/) ซึ่งจากที่ทดลองเวอร์ชั่นล่าสุด ก็มีสามารถเลือก mysql 5 ได้ เพราะฉะนั้นเวอร์ชั่นล่าสุดที่มี mysql 5 ให้เลือกลงได้นั้นคือ mysql-apt-config 0.8.15 ให้ทำการดาวน์โหลดตามนี้:

```sh
wget http://repo.mysql.com/mysql-apt-config_0.8.15-1_all.deb
```

จากนั้นก็ติดตั้งด้วยคำสั่ง:

```sh
sudo dpkg -i mysql-apt-config_0.8.15-1_all.deb
```

แล้วจะมีหน้าจอมาให้เลือกเวอร์ชั่นของ ubuntu ให้เราเลือก `bionic` เพราะจากที่ลองเวอร์ชั่นนี้จะมี mysql 5 ให้เลือก:

![](https://i.imgur.com/iAcf4V4.png)

จากนั้นเลือกเมนูแรก ซึ่งจะเห็นว่าเป็นเวอร์ชั่น 8 อยู่:

![](https://i.imgur.com/S2T4kYc.png)

เมื่อเข้ามาแล้วจะเห็น mysql 5.7 ให้เลือก ก็เลือกเลย:

![](https://i.imgur.com/fzvtwTA.png)

จะเห็นกว่าตอนนี้เราเลือกที่จะลงเวอร์ชั่น 5.7 แล้ว จากนั้นให้เลือก `Ok`:

![](https://i.imgur.com/hj4XDiA.png)

เมื่อเสร็จแล้วมันจะไปสร้าง apt source ให้ใน **/etc/apt/sources.list.d/**:

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
Get:7 http://th.archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [1,790 kB]
Get:8 http://security.ubuntu.com/ubuntu jammy-security/main amd64 Packages [1,583 kB]
Get:9 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 Packages [883 kB]
Get:10 http://th.archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [1,101 kB]
Get:11 http://th.archive.ubuntu.com/ubuntu jammy-updates/universe Translation-en [255 kB]
Reading package lists... Done
W: GPG error: http://repo.mysql.com/apt/ubuntu bionic InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B7B3B788A8D3785C
E: The repository 'http://repo.mysql.com/apt/ubuntu bionic InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

![](https://i.imgur.com/aenH6ao.png)

ให้ดูที่บรรทัดที่มี `NO_PUBKEY`:

```sh
W: GPG error: http://repo.mysql.com/apt/ubuntu bionic InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B7B3B788A8D3785C
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
Hit:1 http://th.archive.ubuntu.com/ubuntu jammy InRelease
Hit:2 http://th.archive.ubuntu.com/ubuntu jammy-updates InRelease
Get:3 http://repo.mysql.com/apt/ubuntu bionic InRelease [20.0 kB]
Hit:4 http://th.archive.ubuntu.com/ubuntu jammy-backports InRelease
Hit:5 https://download.docker.com/linux/ubuntu jammy InRelease
Hit:6 http://security.ubuntu.com/ubuntu jammy-security InRelease
Get:7 http://repo.mysql.com/apt/ubuntu bionic/mysql-5.7 Sources [926 B]
Get:8 http://repo.mysql.com/apt/ubuntu bionic/mysql-apt-config amd64 Packages [566 B]
Get:9 http://repo.mysql.com/apt/ubuntu bionic/mysql-5.7 amd64 Packages [5,676 B]
Get:10 http://repo.mysql.com/apt/ubuntu bionic/mysql-tools amd64 Packages [8,360 B]
Fetched 35.6 kB in 2s (22.2 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
7 packages can be upgraded. Run 'apt list --upgradable' to see them.
W: http://repo.mysql.com/apt/ubuntu/dists/bionic/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
```

![](https://i.imgur.com/md1fs8S.png)

ยังไม่จบ เราต้องแก้การแจ้งเตือนนี้เสียก่อน:

```sh
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

/etc/apt/trusted.gpg.d/ubuntu-keyring-2012-cdimage.gpg
------------------------------------------------------
pub   rsa4096 2012-05-11 [SC]
      8439 38DF 228D 22F7 B374  2BC0 D94A A3F0 EFE2 1092
uid           [ unknown] Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>

/etc/apt/trusted.gpg.d/ubuntu-keyring-2018-archive.gpg
------------------------------------------------------
pub   rsa4096 2018-09-17 [SC]
      F6EC B376 2474 EDA9 D21B  7022 8719 20D1 991B C93C
uid           [ unknown] Ubuntu Archive Automatic Signing Key (2018) <ftpmaster@ubuntu.com>
```

ในส่วนนี้เราจะเจอ keys ของ MySQL Release Engineering อยู่ 2 ตัว เราจะไม่สนใจตัวที่ expired ให้เราดูที่ unknown:

```sh
/etc/apt/trusted.gpg
--------------------
pub   dsa1024 2003-02-03 [SCA] [expired: 2022-02-16]
      A4A9 4068 76FC BD3C 4567  70C8 8C71 8D3B 5072 E1F5
uid           [ expired] MySQL Release Engineering <mysql-build@oss.oracle.com>

pub   rsa4096 2023-10-23 [SC] [expires: 2025-10-22]
      BCA4 3417 C3B4 85DD 128E  C6D4 B7B3 B788 A8D3 785C
uid           [ unknown] MySQL Release Engineering <mysql-build@oss.oracle.com>
sub   rsa4096 2023-10-23 [E] [expires: 2025-10-22]
```

เราจะเห็นชุดของ keys ดังนี้ `BCA4 3417 C3B4 85DD 128E  C6D4 B7B3 B788 A8D3 785C` เราจะใช้แค่ 8 ตัวสุดท้ายคือ `A8D3 785C` (ไม่เอาช่องว่าง) ไปเพิ่มลงใน GPG key โดยสร้างเป็นไฟล์อยู่ภายใต้ `/etc/apt/trusted.gpg.d` directory:

```sh
$ sudo apt-key export A8D3785C | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/mysql.gpg
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
```

เสร็จแล้วก็ `apt update` จบพบว่าแจ้งเตือนต่าง ๆ หายไปแล้ว:

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
7 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

ต่อไปเราจะ install mysql กันแล้ว แต่ว่า ubuntu ก็ยังเลือกเวอร์ชั่น 8 ให้อยู่:

```sh
$ sudo apt install mysql-server
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libcgi-fast-perl libcgi-pm-perl libclone-perl libencode-locale-perl libevent-pthreads-2.1-7 libfcgi-bin libfcgi-perl libfcgi0ldbl libhtml-parser-perl
  libhtml-tagset-perl libhtml-template-perl libhttp-date-perl libhttp-message-perl libio-html-perl liblwp-mediatypes-perl libmecab2 libprotobuf-lite23
  libtimedate-perl liburi-perl mecab-ipadic mecab-ipadic-utf8 mecab-utils mysql-client-8.0 mysql-client-core-8.0 mysql-common mysql-server-8.0
  mysql-server-core-8.0
Suggested packages:
  libdata-dump-perl libipc-sharedcache-perl libbusiness-isbn-perl libwww-perl mailx tinyca
The following NEW packages will be installed:
  libcgi-fast-perl libcgi-pm-perl libclone-perl libencode-locale-perl libevent-pthreads-2.1-7 libfcgi-bin libfcgi-perl libfcgi0ldbl libhtml-parser-perl
  libhtml-tagset-perl libhtml-template-perl libhttp-date-perl libhttp-message-perl libio-html-perl liblwp-mediatypes-perl libmecab2 libprotobuf-lite23
  libtimedate-perl liburi-perl mecab-ipadic mecab-ipadic-utf8 mecab-utils mysql-client-8.0 mysql-client-core-8.0 mysql-common mysql-server mysql-server-8.0
  mysql-server-core-8.0
0 upgraded, 28 newly installed, 0 to remove and 7 not upgraded.
Need to get 29.6 MB of archives.
After this operation, 243 MB of additional disk space will be used.
Do you want to continue? [Y/n] n
Abort.
```

ให้เรายกเลิกการติดตั้งไปก่อน

เราจะต้องให้ apt ไปเรียก package จาก MySQL repository โดยเราต้องกำหนดลำดับก่อน โดยจะกำหนดใน `/etc/apt/preferences.d/mysql` ดังนี้:

```sh
$ cat /etc/apt/preferences.d/mysql
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

ทำตามขั้นตอนก็จะเห็นว่า mysql ที่ระบบลงให้เป็นเวอร์ชั่น 5 แล้ว:

```sh
$ mysql --version
mysql  Ver 14.14 Distrib 5.7.42, for Linux (x86_64) using  EditLine wrapper
```

ตรวจสอบ mysql package ที่ลงไป:

```sh
$ dpkg -l | grep mysql
ii  mysql-apt-config                       0.8.15-1                                all          Auto configuration for MySQL APT Repo.
ii  mysql-client                           5.7.42-1ubuntu18.04                     amd64        MySQL Client meta package depending on latest version
ii  mysql-common                           5.7.42-1ubuntu18.04                     amd64        MySQL Common
ii  mysql-community-client                 5.7.42-1ubuntu18.04                     amd64        MySQL Client
ii  mysql-community-server                 5.7.42-1ubuntu18.04                     amd64        MySQL Server
ii  mysql-server                           5.7.42-1ubuntu18.04                     amd64        MySQL Server meta package depending on latest version
```

และตรวจดู status ของ mysql:

```sh
$ systemctl status mysql
● mysql.service - MySQL Community Server
     Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-07-09 04:19:07 UTC; 2min 54s ago
    Process: 4164 ExecStartPre=/usr/share/mysql/mysql-systemd-start pre (code=exited, status=0/SUCCESS)
    Process: 4203 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid (code=exited, status=0/SUCCESS)
   Main PID: 4205 (mysqld)
      Tasks: 27 (limit: 2219)
     Memory: 171.8M
        CPU: 542ms
     CGroup: /system.slice/mysql.service
             └─4205 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
```

อย่างสุดท้าย เราต้องระวังการ upgrade จาก ubuntu ซึ่งอาจจะ upgrade mysql 5 ไปเป็น 8 ให้เราเองโดยที่เราไม่ตั้งใจ:

```sh
$ sudo apt upgrade
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following packages have been kept back:
  python3-update-manager ubuntu-minimal ubuntu-server ubuntu-server-minimal ubuntu-standard update-manager-core
The following packages will be upgraded:
  mysql-apt-config
1 upgraded, 0 newly installed, 0 to remove and 6 not upgraded.
Need to get 18.2 kB of archives.
After this operation, 0 B of additional disk space will be used.
Do you want to continue? [Y/n] n
Abort.
```

จะเห็นว่ามีการจะอัพเกรด `mysql-apt-config` เราจะ hold การ upgrade ไปก่อนด้วย:

```sh
sudo apt-mark hold mysql-apt-config
```
จบ

## References

- https://www.claudiokuenzler.com/blog/991/install-mysql-5.7-on-ubuntu-20.04-focal-avoid-8.0-packages
- https://chrisjean.com/fix-apt-get-update-the-following-signatures-couldnt-be-verified-because-the-public-key-is-not-available/
- https://itsfoss.com/key-is-stored-in-legacy-trusted-gpg/