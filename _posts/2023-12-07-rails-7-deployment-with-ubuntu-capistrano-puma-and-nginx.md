---
layout: post
title: Rails 7 Deployment with Ubuntu, Capistrano, Puma and Nginx
categories: Posts
tags: rails rails 7 deploy ubuntu capistrano puma nginx
date: 2023-12-07 09:47 +0700
---

ในที่นี้เราจะทำการ deploy Rails ไปที่ Ubuntu server กัน โดยเราจะใช้ stack ดังนี้:

- Ubuntu 22.04 LTS
- Ruby 3.2.2
- Rails 7.0.8
- Puma 5
- Capistrano

> มีโพสต์ที่ใหม่กว่าและครอบคลุมกว่า — [How to deploy Rails 7.2 from scratch with Capistrano, Puma and Nginx on Ubuntu 24.04](/posts/how-to-deploy-rails-7-2-from-scratch-with-capistrano-puma-and-nginx-on-ubuntu-24-04/) ใช้ Puma 6 + Ubuntu 24.04 พร้อม Redis และ Let's Encrypt SSL ครบ ส่วน post นี้ scope ที่ initial deploy ของ Rails 7.0 + Puma 5 โดยเฉพาะ
{: .prompt-info }

## Prerequisites

ฝั่ง local:

- Ruby 3.2.2 (พร้อม Rails 7.0.8)
- SSH client พร้อม key

ฝั่ง server:

- Ubuntu 22.04 LTS
- root access ผ่าน SSH

## Setup Ubuntu

หลังจากติดตั้ง Ubuntu server เรียบร้อยแล้วเราจะสามารถ login เป็น root (หรือชื่ออะไรก็ตามที่ใส่ตอนลง Ubuntu) เข้า server ได้ด้วยคำสั่งนี้ โดยเปลี่ยน `1.2.3.4` เป็น IP ของเครื่อง

```sh
ssh root@1.2.3.4
```
{: file="Local Machine"}

พอ login เข้าได้แล้วก็ทำการอัพเดทระบบสักนิดนึง

```sh
sudo apt update && sudo apt upgrade -y && sudo apt-get autoremove && sudo reboot
```
{: file="root@1.2.3.4"}

ติดตั้ง `net-tools` เพื่อดู IP ของ server

```sh
sudo apt install net-tools
```
{: file="root@1.2.3.4"}

ติดตั้ง `vim` เป็น text editor (แล้วแต่ถนัด)

```sh
sudo apt install vim
```
{: file="root@1.2.3.4"}

### Creating a Deploy User

สร้าง deploy user เพื่อใช้สำหรับลง เราจะไม่ใช้ root เพื่อความปลอดภัย

login เข้า root แล้วสร้าง user deploy และเพิ่มเข้า group sudo

```sh
sudo adduser deploy
sudo adduser deploy sudo
exit
```
{: file="root@1.2.3.4"}

จากนั้นเราจะเพิ่ม SSH key ของเครื่องเราไปที่ server เพื่อจะได้ login server ได้ง่าย ๆ โดยเครื่องมือที่เราจะใช้ชื่อ `ssh-copy-id` ถ้าเราใช้ Mac เราสามารถ `brew install ssh-copy-id` ได้เลย

```sh
ssh-copy-id root@1.2.3.4
ssh-copy-id deploy@1.2.3.4
```
{: file="Local Machine"}

หลังจากนี้เราจะ login เข้า deploy เพื่อทำขั้นตอนต่อไป

## Installing Ruby

เริ่มจากลง package ที่จำเป็นสำหรับการ compile Ruby เสียก่อน

```sh
sudo apt install -y git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates
```
{: file="deploy@1.2.3.4"}

ในที่นี้เราจะใช้ rbenv เป็นตัวจัดการเวอร์ชั่นของ Ruby

```sh
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
git clone https://github.com/rbenv/rbenv-vars.git ~/.rbenv/plugins/rbenv-vars
exec $SHELL
rbenv install 3.2.2
rbenv global 3.2.2
```
{: file="deploy@1.2.3.4"}

> ในชุดคำสั่งด้านบนติดตั้ง 3 ตัว — `rbenv` (Ruby version manager), `ruby-build` (compile Ruby แต่ละเวอร์ชั่น), และ [`rbenv-vars`](https://github.com/rbenv/rbenv-vars) (plugin ที่อ่านไฟล์ `.rbenv-vars` ใน app folder แล้ว load เป็น env var ตอน Ruby รัน — เราจะใช้ตอนตั้ง `SECRET_KEY_BASE` ในขั้นตอนถัดไป)
{: .prompt-info }

เสร็จแล้วก็ตรวจสอบสักนิดว่า rbenv ใช้งานได้แล้ว

```sh
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-doctor | bash
```
{: file="deploy@1.2.3.4"}

ทำการอัพเดท gem ของระบบ

```sh
gem update --system
```
{: file="deploy@1.2.3.4"}

และ

```sh
gem install bundler
```
{: file="deploy@1.2.3.4"}

## Installing NGINX

เราจะใช้ Nginx เป็น webserver สามารถติดตั้งได้ตามขั้นตอนปกติ

```sh
sudo apt install nginx
```
{: file="deploy@1.2.3.4"}

หลังจากนั้นเราจะลบ default ที่ติดมาตอนลงออก

```sh
sudo rm /etc/nginx/sites-enabled/default
```
{: file="deploy@1.2.3.4"}

และทำการ restart nginx

```sh
sudo service nginx restart
```
{: file="deploy@1.2.3.4"}

## Create Rails App

เราจะสร้าง app แบบง่าย ๆ เพื่อเป็นการทดสอบการ deploy

```sh
rails _7.0.8_ new appname
```
{: file="Local Machine"}

จากนั้นก็เอา app นี้ขึ้น GitHub

## Setup SSH keys

ในการ deploy เราจะต้องมีที่เก็บ code ไว้ ในที่นี้เราจะใช้ GitHub ก่อนอื่นเราต้องเช็คก่อนเราสามารถต่อกับ GitHub ได้

```sh
ssh -T git@github.com
```
{: file="deploy@1.2.3.4"}

ก่อนอื่นสร้าง SSH key (แนะนำ ed25519 เพราะเร็วกว่า ปลอดภัยกว่า และ key สั้นกว่า RSA):

```sh
ssh-keygen -t ed25519 -C "deploy@appname"
```
{: file="deploy@1.2.3.4"}

> ถ้า server เก่ามากที่ยังรองรับเฉพาะ RSA ใช้ `ssh-keygen -t rsa -b 4096` แทน
{: .prompt-tip }

ให้เว้นว่างไม่ต้องใส่ password

แล้วเราสามารถดู key ที่สร้างด้วยคำสั่งนี้

```sh
cat ~/.ssh/id_ed25519.pub
```
{: file="deploy@1.2.3.4"}

ทำตามขั้นตอนนี้ [Set up deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#set-up-deploy-keys) จากนั้นรันคำสั่ง `ssh -T git@github.com` อีกครั้ง ต้องขึ้นว่าสำเร็จ เป็นอันจบ

## Setup Production Variables

ก่อนที่จะ deploy ครั้งแรกให้เราเข้าไปสร้างโฟลเดอร์ตาม location ที่กำหนดตาม `set :deploy_to` ใน `config/deploy.rb` จากนั้นสร้างไฟล์ `.rbenv-vars`

```sh
$ mkdir [DEPLOY_TO]
$ cd [DEPLOY_TO]
$ vim .rbenv-vars
```
{: file="deploy@1.2.3.4"}

จากนั้นเพิ่มตามนี้ลงในไฟล์

```
SECRET_KEY_BASE=<random sequence> # use local `rails secret`
RAILS_ENV=production
```
{: file=".rbenv-vars"}

`SECRET_KEY_BASE` เราสามารถสร้างได้ด้วยคำสั่งนี้บนเครื่อง local

```sh
rails secret
```
{: file="Local Machine"}

## Setup Capistrano

เราจะใช้ Capistrano สำหรับ automate deployments เพื่อให้การ deploy ทำได้ง่าย ๆ

เพิ่ม gem ที่จำเป็นใน `Gemfile`:

```ruby
group :development do
  gem "bcrypt_pbkdf"
  gem "capistrano"
  gem "capistrano3-puma", "~> 5"
  gem "capistrano-rails"
  gem "ed25519"
end
```
{: file="Gemfile"}

> อธิบายแต่ละ gem:
>
> - **`capistrano`** — core ของ Capistrano (SSH automation framework)
> - **`capistrano3-puma "~> 5"`** — task สำหรับ Puma 5 รวมถึง `puma:config`, `puma:systemd:config`, `puma:nginx_config` (Puma 6 ใช้ `capistrano3-puma "~> 6"` ซึ่งรวมเป็น `puma:install` ตัวเดียว)
> - **`capistrano-rails`** — task เฉพาะของ Rails (`assets:precompile`, `migrate`) และ default `linked_files`/`linked_dirs` ของ Rails project
> - **`bcrypt_pbkdf` + `ed25519`** — net-ssh ต้องการ 2 gem นี้ตอนใช้ ED25519 SSH key (ที่เราสร้างไว้ในขั้นตอนก่อนหน้า) ถ้าไม่มีจะ fall back ไป RSA หรือ error ได้
{: .prompt-info }

ติดตั้ง gem ที่เพิ่มเข้ามา

```sh
bundle
```
{: file="Local Machine"}

ใช้ Capistrano สร้างไฟล์ที่จำเป็นในการ deploy

```sh
$ cap install
mkdir -p config/deploy
create config/deploy.rb
create config/deploy/staging.rb
create config/deploy/production.rb
mkdir -p lib/capistrano/tasks
create Capfile
Capified
```
{: file="Local Machine"}

จากนั้นแก้ไฟล์ `Capfile` ตามนี้

```ruby
# Load DSL and set up stages
require "capistrano/setup"

# Include default deployment tasks
require "capistrano/deploy"

# Load the SCM plugin appropriate to your project:
require "capistrano/scm/git"
install_plugin Capistrano::SCM::Git

# Include tasks from other gems included in your Gemfile
require "capistrano/rbenv"
require "capistrano/bundler"
require "capistrano/rails/assets"
require "capistrano/rails/migrations"

require "capistrano/puma"
install_plugin Capistrano::Puma
install_plugin Capistrano::Puma::Systemd
require "capistrano/puma/nginx"
install_plugin Capistrano::Puma::Nginx

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```
{: file="Capfile"}

จากนั้นแก้ไขไฟล์ `config/deploy.rb` ตามนี้

```ruby
# config valid for current version and patch releases of Capistrano
lock "~> 3.18.0"

set :application, "[APP_NAME]"
set :user, "[DEPLOY_USER]"
set :repo_url, "[GIT SSH ADDRESS: git@github.com:username/appname.git]"

set :branch, "main"

# Default deploy_to directory is /var/www/my_app_name
set :deploy_to, "/home/#{fetch :user}/#{fetch :application}"

# Files that get symlinked from shared/ to each release (persisted across deploys)
append :linked_files, "config/database.yml", "config/master.key"

# Directories that get symlinked from shared/ to each release
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "vendor/bundle", ".bundle", "storage"

set :rbenv_ruby, "3.2.2"
```
{: file="config/deploy.rb"}

> `linked_files` คือไฟล์ที่ไม่ได้อยู่ใน git แต่ต้องมีบน server (`master.key`, `database.yml`) Capistrano จะ symlink จาก `shared/` เข้าไปในทุก release แทนการ copy ใหม่ทุกครั้ง
>
> `linked_dirs` ทำหน้าที่คล้ายกันแต่กับ folder — `log`, `tmp/pids`, `storage` ของ Active Storage ฯลฯ ที่อยากเก็บข้ามไปทุก release
{: .prompt-info }

และสุดท้ายแก้ไขไฟล์ `config/deploy/production.rb`

```ruby
server "[PRODUCTION_IP]", user: "[DEPLOY_USER]", roles: %w[app db web]
```
{: file="config/deploy/production.rb"}

## Deploy

เริ่มแรกให้รัน `deploy:check` ก่อน เพื่อเป็นการสร้างโครงสร้างของโฟลเดอร์บนเครื่องที่เราจะ deploy

```sh
cap production deploy:check
```
{: file="Local Machine"}

แน่นอนว่ามาถึงตรงนี้จะมี error ว่าไม่มีไฟล์อย่าง `database.yml`, `master.key` ซึ่งเป็นรายชื่อไฟล์ที่เรากำหนดใน `linked_files` นั่นแหละ ให้เราทำการ copy ไปไว้ในเครื่อง deploy ก่อน

```sh
scp config/database.yml [DEPLOY_USER]@[PRODUCTION_IP]:[DEPLOY_TO]/shared/config/database.yml
scp config/master.key [DEPLOY_USER]@[PRODUCTION_IP]:[DEPLOY_TO]/shared/config/master.key
```
{: file="Local Machine"}

จากนั้นให้รันคำสั่งนี้

```sh
cap production puma:config
```
{: file="Local Machine"}

จากนั้นเป็นการ config ให้ใช้ Puma ผ่าน [systemd](https://www.freedesktop.org/wiki/Software/systemd/)

```sh
cap production puma:systemd:config
cap production puma:systemd:enable
```
{: file="Local Machine"}

> ใน `capistrano3-puma 5` task ของ Puma + systemd + nginx ต้องรันแยกตามลำดับ: `puma:config` → `puma:systemd:config` → `puma:systemd:enable` → `puma:nginx_config` ส่วน Puma 6 (`capistrano3-puma 6`) รวมเป็น `puma:install` ตัวเดียว ดูที่[โพสต์ Rails 7.2](/posts/how-to-deploy-rails-7-2-from-scratch-with-capistrano-puma-and-nginx-on-ubuntu-24-04/)
{: .prompt-info }

> ในขั้นตอนนี้อาจจะมีการขอสิทธิ์ sudo ในตอนรันคำสั่ง ซึ่งจะมี error ประมาณนี้:
>
> ```
> 01 sudo /bin/systemctl restart puma_appname_production
> 01 sudo
> 01 :
> 01 a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
> ```
>
> ให้เรา login เข้า deploy user แล้วทำตามนี้:
>
> ```sh
> sudo visudo
> ```
> {: file="deploy@1.2.3.4"}
>
> จากนั้นเพิ่ม `deploy  ALL=(ALL) NOPASSWD:ALL` หลังบรรทัด `%sudo   ALL=(ALL:ALL) ALL`
>
> ```
> # Allow members of group sudo to execute any command
> %sudo   ALL=(ALL:ALL) ALL
>
> deploy  ALL=(ALL) NOPASSWD:ALL
> ```
> {: file="/etc/sudoers" }
{: .prompt-danger }

ขั้นต่อไปรันคำสั่งนี้เพื่อเป็นการ config nginx

```sh
cap production puma:nginx_config
```
{: file="Local Machine"}

เมื่อเสร็จแล้วเราก็ต้อง restart nginx อีกสักรอบ

```sh
sudo service nginx restart
```
{: file="deploy@1.2.3.4"}

จากนั้นให้ทำการ deploy ได้เลย

```sh
cap production deploy
```
{: file="Local Machine"}

แล้วไปที่ browser แล้วไปที่ `PRODUCTION_IP`

> บางทีเราอาจจะเจอกับ 503 bad gateway ซึ่งเมื่อดู log แล้วจะเป็นการฟ้องเรื่อง permission
>
> ```sh
> tail -f /var/log/nginx/error.log
>
> ...failed (13: Permission denied)...
> ```
> {: file="deploy@1.2.3.4"}
>
> ให้เราทำการแก้ permission ของ deploy user ได้ด้วยคำสั่งนี้
>
> ```sh
> cd /home
> sudo chmod o=rx $USER/
> ```
> {: file="deploy@1.2.3.4"}
{: .prompt-danger }

> และบางทีอาจจะเจอ 500 Internal Server Error ให้ไปดู log ของ `puma_error.log` หากเจอว่า
>
> ```
> Permission denied @ rb_io_reopen - /home/.../shared/log/puma_access.log (Errno::EACCES)
> ```
>
> ให้ทำการแก้ไข permission ของโฟลเดอร์ appname เสียก่อน
>
> ```sh
> sudo chown $USER:$USER -R /home/$USER/appname/
> ```
> {: file="deploy@1.2.3.4"}
>
> จากนั้นก็ทำการ restart Puma
>
> ```sh
> cap production puma:restart
> ```
> {: file="Local Machine"}
{: .prompt-danger }

## Conclusion

การ deploy แบบเริ่มต้นที่จำเป็นมีเท่านี้ ซึ่งในที่นี้เรายังขาดการติดตั้ง database และ background process ซึ่งส่วนตัวแล้วไม่ยากเท่าไรแล้วหลังเราผ่านการ deploy แบบเริ่มต้นมาแล้ว

ต่อจากนี้ดูได้ที่:

- [How to install MySQL 5.7 on Ubuntu 22.04](/posts/how-to-install-mysql-5-7-on-ubuntu-22-04/) — ติดตั้ง database
- [Deploy Solid Queue with Capistrano](/posts/deploy-solid-queue-with-capistrano/) — background job
- [Capistrano support for propshaft](/posts/capistrano-support-for-propshaft/) — แก้ deploy error เมื่อใช้ Propshaft (Rails 7.1+)
- [How to deploy Rails 7.2 from scratch with Capistrano, Puma and Nginx on Ubuntu 24.04](/posts/how-to-deploy-rails-7-2-from-scratch-with-capistrano-puma-and-nginx-on-ubuntu-24-04/) — เวอร์ชั่นที่ใหม่กว่าและครอบคลุมกว่า (Puma 6 + SSL)

## References

- [Capistrano documentation](https://capistranorb.com/)
- [capistrano3-puma](https://github.com/seuros/capistrano-puma)
- [rbenv-vars plugin](https://github.com/rbenv/rbenv-vars)
- [GoRails — Deploy Ruby on Rails to Ubuntu 22.04](https://gorails.com/deploy/ubuntu/22.04)
- [Rails 7 production deploy from scratch (Ubuntu 22.04)](https://dev.to/1klap/rails-7-production-deploy-from-scratch-ubuntu-2204-edition-pm7)
- [Deploying a Rails App on Ubuntu 20.04 — Matthew Hoelter](https://www.matthewhoelter.com/2020/11/10/deploying-ruby-on-rails-for-ubuntu-2004.html)
