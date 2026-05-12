---
layout: post
title: Deploy Solid Queue with Capistrano
categories:
- Posts
tags:
- rails
- deploy
- capistrano
date: 2025-01-03 11:52 +0700
---

[Solid Queue](https://github.com/rails/solid_queue) เป็น background job backend ของ Rails 8 ที่เก็บงานไว้ใน database โดยตรง ไม่ต้องพึ่ง Redis หรือ Sidekiq สำหรับการ deploy ผ่าน Capistrano วิธีที่สะอาดที่สุดคือรัน Solid Queue เป็น systemd user service ของ deploy user ไม่ต้องใช้ sudo และ lifecycle ผูกกับ deploy hook ของ Capistrano ได้ตรงๆ โพสต์นี้พาตั้งค่าตั้งแต่ติดตั้ง gem จนเชื่อมเข้ากับ Capistrano deploy flow

## Prerequisites

- Rails 8 (หรือ Rails 7.1+ ที่ติดตั้ง `solid_queue` gem แล้ว)
- Capistrano deploy flow ที่ใช้งานได้อยู่
- Server ที่ใช้ systemd (Ubuntu 22.04+, Debian 12+, ฯลฯ)
- Deploy user บน server (เช่น `deploy`) ที่ใช้ rbenv หรือ Ruby manager อื่น

## ติดตั้ง Solid Queue ใน Rails

ถ้ายังไม่ได้ตั้งค่าใน app:

```sh
bin/rails solid_queue:install
bin/rails db:migrate
```

แล้วตั้ง queue adapter ใน `config/application.rb` หรือ `config/environments/production.rb`:

```ruby
config.active_job.queue_adapter = :solid_queue
```
{: file="config/environments/production.rb"}

## Enable lingering ให้ deploy user

systemd user service จะถูก start เมื่อ user มี session อยู่ และจะดับเมื่อ user logout การ enable lingering ทำให้ service ทำงานต่อเนื่องแม้ไม่มี session — จำเป็นสำหรับ background worker บน production

รันบน server ด้วย sudo ครั้งเดียว:

```sh
sudo loginctl enable-linger <deploy user>
```

> ถ้าลืมขั้นตอนนี้ Solid Queue จะดับทันทีหลัง deploy เพราะ Capistrano logout ออก session จะหายไปด้วย
{: .prompt-warning }

## สร้าง systemd user unit

วาง unit file ไว้ใน home ของ deploy user:

```ini
[Unit]
Description=solid_queue for app
After=syslog.target network.target

[Service]
Type=simple
Environment=RAILS_ENV=production
WorkingDirectory=/home/<deploy user>/<app name>/current
ExecStart=/home/<deploy user>/.rbenv/bin/rbenv exec bundle exec rake solid_queue:start
ExecReload=/bin/kill -TSTP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID

Environment=MALLOC_ARENA_MAX=2

RestartSec=1
Restart=on-failure

SyslogIdentifier=solid_queue

[Install]
WantedBy=default.target
```
{: file="~/.config/systemd/user/solid_queue.service"}

อธิบาย directive ที่สำคัญ:

- **`Type=simple`** — systemd ถือว่า process หลักคือคำสั่งที่ `ExecStart` รัน ไม่ต้อง fork
- **`WorkingDirectory=.../current`** — ชี้ไปที่ symlink `current` ของ Capistrano เพื่อให้ทุก deploy ใช้ unit เดิมโดยไม่ต้องแก้
- **`ExecStart=... rbenv exec bundle exec rake solid_queue:start`** — ผ่าน `rbenv exec` เพื่อให้ได้ Ruby/gem version ที่ตรงกับ `.ruby-version` ของ app
- **`MALLOC_ARENA_MAX=2`** — ลด memory fragmentation ของ glibc malloc ที่ Ruby มักเจอใน multi-thread worker (default จะสร้าง arena ตามจำนวน CPU ทำให้ RSS บวมเกินจำเป็น)
- **`ExecReload=kill -TSTP`** — ส่ง `SIGTSTP` ให้ Solid Queue เริ่ม graceful shutdown (หยุดรับงานใหม่ ทำงานปัจจุบันให้จบ)
- **`ExecStop=kill -TERM`** — ส่ง `SIGTERM` ให้หยุดทันทีหลัง graceful หมดเวลา
- **`Restart=on-failure` + `RestartSec=1`** — restart อัตโนมัติเมื่อ process ตายผิดปกติ (ไม่ restart ถ้า exit 0)
- **`WantedBy=default.target`** — ของ systemd user mode (เทียบเท่า `multi-user.target` ในระดับ system)

## Reload, enable, และ start

หลังสร้างไฟล์แล้ว สั่ง systemd อ่าน unit ใหม่ enable ให้รัน auto เมื่อ user lingering start และ start ครั้งแรก:

```sh
systemctl --user daemon-reload
systemctl --user enable solid_queue.service
systemctl --user start solid_queue.service
```

## ผูก lifecycle เข้ากับ Capistrano

เพิ่ม task และ hook ใน `config/deploy.rb`:

```ruby
set :solid_queue_systemd_unit_name, "solid_queue.service"

namespace :solid_queue do
  desc "Quiet solid_queue (start graceful termination)"
  task :quiet do
    on roles(:app) do
      execute :systemctl, "--user", "kill", "-s", "SIGTERM", fetch(:solid_queue_systemd_unit_name), raise_on_non_zero_exit: false
    end
  end

  desc "Stop solid_queue (force immediate termination)"
  task :stop do
    on roles(:app) do
      execute :systemctl, "--user", "kill", "-s", "SIGQUIT", fetch(:solid_queue_systemd_unit_name), raise_on_non_zero_exit: false
    end
  end

  desc "Start solid_queue"
  task :start do
    on roles(:app) do
      execute :systemctl, "--user", "start", fetch(:solid_queue_systemd_unit_name)
    end
  end

  desc "Restart solid_queue"
  task :restart do
    on roles(:app) do
      execute :systemctl, "--user", "restart", fetch(:solid_queue_systemd_unit_name)
    end
  end
end

# SolidQueue hooks
after "deploy:starting", "solid_queue:quiet"
after "deploy:updated", "solid_queue:stop"
after "deploy:published", "solid_queue:start"
after "deploy:failed", "solid_queue:restart"
```
{: file="config/deploy.rb"}

ลำดับ hook อ่านยังไง:

- **`deploy:starting` → `quiet`** — ตั้งแต่ deploy เริ่ม ส่ง `SIGTERM` ให้ worker เริ่ม graceful shutdown หยุดหยิบงานใหม่ ทำงานปัจจุบันให้จบ ขณะ Capistrano เตรียม release ใหม่อยู่
- **`deploy:updated` → `stop`** — หลัง symlink `current` ชี้ไปที่ release ใหม่แล้ว ส่ง `SIGQUIT` หยุด worker สนิท เพราะโค้ดเดิมไม่ต้องใช้แล้ว
- **`deploy:published` → `start`** — start worker ใหม่ขึ้นมาด้วยโค้ดใหม่
- **`deploy:failed` → `restart`** — ถ้า deploy ล้ม สั่ง restart เพื่อให้กลับสู่สถานะปกติ (ไม่ทิ้ง worker ค้างไว้แบบ stopped)

`raise_on_non_zero_exit: false` ที่ `quiet` และ `stop` กันกรณี service ไม่ได้รันอยู่ (เช่น deploy ครั้งแรก) จะได้ไม่ทำให้ deploy ล้มทั้งกระบวน

## Verify

เช็ค status:

```sh
systemctl --user status solid_queue
```

ดู log แบบ tail:

```sh
journalctl --user -u solid_queue -f
```

แล้วลอง enqueue job จาก Rails console:

```ruby
SomeJob.perform_later
```

ถ้าทุกอย่างทำงาน job จะถูกหยิบไปทำในเวลาไม่กี่วินาที และ log จะแสดงใน `journalctl`

## Troubleshooting

**Service ดับทุกครั้งที่ deploy เสร็จ**
ส่วนใหญ่ลืม `loginctl enable-linger` ตรวจด้วย `loginctl show-user <deploy user> | grep Linger` ถ้าได้ `Linger=no` ให้ enable ใหม่

**`bundle: command not found` ใน `journalctl`**
systemd user service ไม่ได้โหลด `.bashrc` หรือ `.zshrc` ของ user ใช้ absolute path ของ `rbenv exec` (ตามตัวอย่าง `ExecStart`) แทนที่จะพึ่ง PATH

**`Failed to connect to bus: No such file or directory` ตอนรัน `systemctl --user` ผ่าน SSH**
ใช้ `loginctl enable-linger` แล้ว ssh ใหม่ หรือใช้ `XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user ...` เป็น workaround ใน Capistrano task

## Conclusion

วิธีนี้ทำให้ Solid Queue มี process manager ที่เชื่อถือได้ (systemd) โดยไม่ต้องเพิ่ม dependency เช่น Foreman, monit, หรือ god และ Capistrano hook ครอบ lifecycle ครบ — quiet ก่อน deploy stop หลัง update start หลัง publish restart ถ้าล้ม

ถ้าย้ายไปใช้ Kamal แทน Capistrano ตัว Solid Queue มักจะรันเป็น role แยก (`bin/jobs`) ใน container ของตัวเอง ซึ่งง่ายกว่ามาก ไม่ต้องยุ่งกับ systemd ของ host เลย — แต่ถ้ายังอยู่บน Capistrano วิธีในโพสต์นี้น่าจะตอบโจทย์เกือบทุกกรณี

## References

- [Solid Queue README](https://github.com/rails/solid_queue)
- [systemd user services (Arch Wiki)](https://wiki.archlinux.org/title/Systemd/User)
- [How I deploy Solid Queue with Capistrano — Rob Zolkos](https://world.hey.com/robzolkos/how-i-deploy-solid-queue-with-capistrano-487b4a31)
