---
layout: post
title: How to deploy Rails 7.2 from scratch with Capistrano, Puma and Nginx on Ubuntu
  24.04
categories: Posts
tags: rails rails 7.2 ubuntu deploy nginx puma capistrano
date: 2024-07-12 16:36 +0700
---
ครั้งก่อนได้เขียนเรื่องนี้ไปรอบหนึ่งแล้ว แต่หลังจากที่ Rails ออกเวอร์ชั่น 7.1 ก็มีอะไรเปลี่ยนแปลงไปนิดหน่อย ประกอบกับตอนนี้ 7.2 ก็ออกตัวเบต้าแล้ว และมี Ubuntu ที่ออกเวอร์ชั่น 24.04 ด้วย แต่หลังจากที่ไปลองเอาที่เขียนก่อนหน้าไปทำการลงดูก็พบว่ามีอะไรเปลี่ยนไปอยู่เหมือนกัน เลยคิดว่าเอามาเขียนอีกรอบดีกว่า

## Prerequisites

ฝั่ง server:

- Ubuntu 24.04 LTS (VirtualBox หรือ VPS provider ใดๆ ก็ได้)
- root access ผ่าน SSH

ฝั่ง local:

- Ruby 3.3.4
- Rails 7.2.0.beta3
- Capistrano 3.19.1
- Puma 6
- Nginx (ติดตั้งบน server ในขั้นตอนถัดไป)

## Configure Production Server

ให้เราติดตั้ง `net-tools` ซะก่อน เพื่อจะได้ดู IP ของ server ได้จาก `ifconfig`:

```sh
sudo apt install net-tools
ifconfig
```
{: file="Remote Server"}

สมมุติว่า IP ที่ได้คือ `1.2.3.4` นะ

พอได้ IP มาแล้วให้ไปใช้เครื่อง local ของเราให้เรา ssh เข้าไปที่ server โดยใช้ชื่อ root หรือชื่ออะไรก็ตามที่กรอกตอนติดตั้ง Ubuntu:

```sh
ssh root@1.2.3.4
```
{: file="Local Machine"}

พอ login ได้แล้ว เราจะทำการอัพเดทระบบกันก่อน:

```sh
sudo apt update && sudo apt upgrade -y && sudo apt-get autoremove && sudo reboot
```
{: file="root@1.2.3.4"}

และทำการติดตั้ง packages ที่จำเป็น:

```sh
sudo apt install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates
```
{: file="root@1.2.3.4"}

### Setup Firewall

หลังจากติดตั้ง Ubuntu เสร็จ ถ้าเป็นเวอร์ชั่น 22.04 นั้นเราสามารถที่จะ ssh เข้าไปได้เลย แต่พอมาเป็น 24.04 กลับทำไม่ได้ ซึ่งเราจะต้องตั้งค่า firewall เสียก่อน:[^setup-firewall]

[^setup-firewall]: [Setting Up and Securing SSH on Ubuntu 22.04: A Comprehensive Guide](https://serverastra.com/docs/Tutorials/Setting-Up-and-Securing-SSH-on-Ubuntu-22.04%253A-A-Comprehensive-Guide)

```sh
sudo apt install openssh-server ufw
sudo ufw status
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo systemctl restart ssh
sudo systemctl status ssh
```
{: file="Remote Server"}

หลังจากเสร็จแล้วเราจะใช้เครื่อง local ในการ remote เข้ามาที่ server ให้เหมือนสภาพจริง ๆ ที่เราจะทำงานกัน

### Create Deploy User

เราจะสร้าง user บน server เพื่อใช้ deploy Rails app ของเรา ในที่นี้ให้ชื่อ `deploy` ซึ่งในความคิดผมนั้น อาจจะเอาชื่อ project ชื่อ app มาใช้เป็นชื่อ user ก็ได้ เพราะว่าใน server ของเรา อาจจะมีหลาย ๆ ตัว deploy ไว้ด้วยกัน เพื่อให้ง่ายต่อการจัดการ

```sh
sudo adduser deploy
sudo adduser deploy sudo
exit
```
{: file="root@1.2.3.4"}

ต่อไปเราจะเพิ่ม SSH key ไปที่ server เพื่อให้สามารถ login ได้อย่างรวดเร็วโดยจะใช้เครื่องมือที่เรียกว่า `ssh-copy-id`

หากใช้เครื่อง Mac จะต้องติดตั้ง `ssh-copy-id` ด้วย Homebrew เสียก่อน `brew install ssh-copy-id`:

```sh
ssh-copy-id root@1.2.3.4
ssh-copy-id deploy@1.2.3.4
```
{: file="Local Machine"}

ตอนนี้เราสามารถที่จะเข้า root และ deploy ได้โดยไม่ต้องใส่ password แล้ว:

```sh
ssh deploy@1.2.3.4
```
{: file="Local Machine"}

แต่ถ้า `ssh-key is not recognized` ขึ้นมาให้รันคำสั่งตามนี้:

```sh
eval $(ssh-agent)
ssh-add
```
{: file="Local Machine"}

## Install Ruby

เราจะใช้ `rbenv` ในการติดตั้ง Ruby โดยทำตามนี้:

```sh
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
git clone https://github.com/rbenv/rbenv-vars.git ~/.rbenv/plugins/rbenv-vars
exec $SHELL
rbenv install 3.3.4
rbenv global 3.3.4
```
{: file="deploy@1.2.3.4"}

> ในชุดคำสั่งด้านบนติดตั้ง 3 ตัว — `rbenv` (Ruby version manager), `ruby-build` (compile Ruby แต่ละเวอร์ชั่น), และ [`rbenv-vars`](https://github.com/rbenv/rbenv-vars) (plugin ที่อ่านไฟล์ `.rbenv-vars` ใน app folder แล้ว load เป็น env var ตอน Ruby รัน — เราจะใช้ตอนตั้ง `SECRET_KEY_BASE` ในขั้นตอนถัดไป)
{: .prompt-info }

ตรวจสอบว่าติดตั้งเรียบร้อยหรือไม่ด้วย `rbenv-doctor`:

```sh
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-doctor | bash
```
{: file="deploy@1.2.3.4"}

ตรวจดูว่า Ruby ที่ลงไปถูกเวอร์ชั่นมั้ย:

```sh
ruby -v
```
{: file="deploy@1.2.3.4"}

ตั้งค่า `.gemrc` เพื่อให้ตอนที่ลง gem จะได้ไม่ต้องลง document มาด้วย (เพื่อความไว):

```sh
echo "gem: --no-document" > ~/.gemrc
```
{: file="deploy@1.2.3.4"}

ติดตั้ง `bundler`:

```sh
gem install bundler
```
{: file="deploy@1.2.3.4"}

หลังจากติดตั้งอาจจะบอกให้อัพเดท:

```sh
gem update --system
```
{: file="deploy@1.2.3.4"}

ดูว่า `bundler` ที่ติดตั้งถูกต้องมั้ย:

```sh
bundle -v
```
{: file="deploy@1.2.3.4"}

## Install NGINX

ทำการติดตั้ง Nginx ตามนี้:

```sh
sudo apt install nginx
```
{: file="deploy@1.2.3.4"}

และเปิด browser ตาม IP:

```sh
open http://1.2.3.4
```
{: file="Local Machine"}

เจอข้อความต้อนรับก็เป็นอันเรียบร้อย:

![](https://i.imgur.com/MY8Jl29.png)

จากนั้นก็ลบ default ของ Nginx ได้เลย:

```sh
sudo rm /etc/nginx/sites-enabled/default
```
{: file="deploy@1.2.3.4"}


## Install Redis

ติดตั้ง Redis เพราะว่า Rails อาจจะต้องการในบางฟีเจอร์ สามารถทำตามขั้นตอนนี้ได้เลย[^redis]:

[^redis]: [How to Install Redis on Ubuntu/Debian and Ensure Auto-Restart on System Reboot](https://medium.com/@kbablu557/how-to-install-redis-on-ubuntu-debian-and-ensure-auto-restart-on-system-reboot-4c9fe675bdb1)

```sh
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt update
sudo apt install redis
```
{: file="deploy@1.2.3.4"}

ดูเวอร์ชั่น:

```sh
$ redis-cli --version
redis-cli 7.2.5
```
{: file="deploy@1.2.3.4"}

ทดสอบว่าทำงานได้:

```sh
$ redis-cli ping
PONG
```
{: file="deploy@1.2.3.4"}

จากนั้นตั้งค่าให้ทำงานตอนเครื่องบูทด้วย:

```sh
sudo systemctl enable redis-server
```
{: file="deploy@1.2.3.4"}

## Create Rails App

เราจะสร้าง app แบบง่าย ๆ เพื่อเป็นการทดสอบการ deploy โดยเป็น Rails 7.2.0.beta2 หากว่ายังไม่ได้ติดตั้งให้ติดตั้งด้วย `gem install rails --pre`:

```sh
rails new my_app
cd my_app
bin/rails g controller main index
```
{: file="Local Machine"}

แล้วกำหนดหน้าแรกใน `config/routes.rb`:

```ruby
root "main#index"
```
{: file="config/routes.rb"}

จากนั้นก็เอา app นี้ขึ้น GitHub ให้เรียบร้อย

## Setup SSH keys

ในขั้นตอนการ deploy แน่นอนว่าเราจะต้องไปดึงโค้ดมาจากที่ที่เก็บไว้สักที่ ในที่นี้เราจะใช้ GitHub เพื่อจะเข้าถึง GitHub เราต้องติดตั้ง SSH key ก่อนเพื่อที่เราจะได้ไม่ต้องใส่พาสเวิร์ดทุกครั้ง[^ssh]:

[^ssh]: [Deploying a Rails App on Ubuntu 20.04 LTS with Capistrano, Nginx, and Puma](https://www.matthewhoelter.com/2020/11/10/deploying-ruby-on-rails-for-ubuntu-2004.html)

```sh
$ ssh -T git@github.com
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
git@github.com: Permission denied (publickey).
```
{: file="deploy@1.2.3.4"}

เริ่มจากสร้าง SSH key ใหม่ (แนะนำ ed25519 เพราะเร็วกว่า ปลอดภัยกว่า และ key สั้นกว่า RSA):

```sh
ssh-keygen -t ed25519 -C "deploy@my_app"
```
{: file="deploy@1.2.3.4"}

> ถ้า server เก่ามากที่ยังรองรับเฉพาะ RSA ใช้ `ssh-keygen -t rsa -b 4096` แทน
{: .prompt-tip }

เมื่อทำตามนี้มาถึงตอนที่ถามพาสเวิร์ดก็ให้ปล่อยว่างไว้ แล้ว SSH key จะถูกสร้างไว้ที่ `~/.ssh/id_ed25519.pub` ซึ่งสามารถเข้าไปดูได้ด้วยคำสั่ง:

```sh
cat ~/.ssh/id_ed25519.pub
```
{: file="deploy@1.2.3.4"}

ไปที่ Settings ของ repository บน GitHub แล้วไปที่เมนู Deploy keys จากนั้นเอา SSH key ที่ได้ใส่ในช่อง Key ได้เลย:

![](https://i.imgur.com/q9XQRQU.png)


ตอนนี้หากเรารันคำสั่ง `ssh -T git@github.com` ก็จะสำเร็จแล้ว

## Setup Capistrano

เราจะใช้ Capistrano สำหรับ automate deployments เพื่อให้การ deploy ทำได้ง่าย ๆ

เพิ่ม gem ที่จำเป็นใน `Gemfile`:

```ruby
group :development do
  gem "capistrano"
  gem "capistrano3-puma", "~> 6.beta.1"
  gem "capistrano-rails"
  gem "capistrano-rbenv"
end
```
{: file="Gemfile"}

อธิบายแต่ละ gem:

- **`capistrano`** — core ของ Capistrano (SSH automation framework)
- **`capistrano3-puma`** — task สำหรับ start/stop/restart Puma ผ่าน systemd รวมถึง `puma:install` ที่ generate systemd unit ให้
- **`capistrano-rails`** — task เฉพาะของ Rails (`assets:precompile`, `migrate`, etc.) และ default `linked_files`/`linked_dirs` สำหรับ Rails project
- **`capistrano-rbenv`** — บอก Capistrano ให้ใช้ Ruby version จาก rbenv ของ deploy user (ตรงกับที่ติดตั้งไว้)

ติดตั้ง gem ที่เพิ่มเข้ามา:

```sh
bundle install
```
{: file="Local Machine"}

ใช้ Capistrano สร้างไฟล์ที่จำเป็นในการ deploy:

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

จากนั้นแก้ไฟล์ `Capfile` ตามนี้:

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

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```
{: file="Capfile"}

จากนั้นแก้ไขไฟล์ `config/deploy.rb` ตามนี้:

```ruby
# config valid for current version and patch releases of Capistrano
lock "~> 3.19.1"

set :application, "[APP_NAME]"
set :user, "[DEPLOY_USER]"
set :repo_url, "[GIT SSH ADDRESS: git@github.com:username/appname.git]"

set :branch, "main"

# Default deploy_to directory is /var/www/my_app_name
set :deploy_to, "/home/#{fetch :user}/#{fetch :application}"

# Files that get symlinked from shared/ to each release (persisted across deploys)
append :linked_files, "config/database.yml", "config/master.key", "config/puma.rb"

# Directories that get symlinked from shared/ to each release
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "vendor/bundle", ".bundle", "storage"

set :rbenv_ruby, "3.3.4"
```
{: file="config/deploy.rb"}

> `linked_files` คือไฟล์ที่ไม่ได้อยู่ใน git แต่ต้องมีบน server (`master.key`, `database.yml`) Capistrano จะ symlink จาก `shared/` เข้าไปในทุก release แทนการ copy ใหม่ทุกครั้ง
>
> `linked_dirs` ทำหน้าที่คล้ายกันแต่กับ folder — `log`, `tmp/pids`, `storage` ของ Active Storage ฯลฯ ที่อยากเก็บข้ามไปทุก release
{: .prompt-info }

และสุดท้ายแก้ไขไฟล์ `config/deploy/production.rb`:

```ruby
server "[PRODUCTION_IP]", user: "[DEPLOY_USER]", roles: %w[app db web worker]
```
{: file="config/deploy/production.rb"}

## Setup Production Variables

ก่อนที่จะ deploy ครั้งแรกให้เราเข้าไปสร้างโฟลเดอร์ตาม location ที่กำหนดตาม `set :deploy_to` ใน `config/deploy.rb` จากนั้นสร้างไฟล์ `.rbenv-vars`:

```sh
$ mkdir [DEPLOY_TO]
$ cd [DEPLOY_TO]
$ vi .rbenv-vars
```
{: file="deploy@1.2.3.4"}

จากนั้นเพิ่มตามนี้ลงในไฟล์:

```
SECRET_KEY_BASE=<random sequence> # use local `rails secret`
RAILS_ENV=production
```
{: file=".rbenv-vars"}

ซึ่ง `SECRET_KEY_BASE` เราสามารถสร้างได้ด้วยคำสั่งนี้บนเครื่อง local:

```sh
rails secret
```
{: file="Local Machine"}

## Deploy

> ตั้งแต่ Rails 7.1 ได้ตั้งค่า `force_ssl` ใน production เป็น `true` เป็นค่าเริ่มต้น ในช่วงแรกที่เรา deploy อาจจะตั้งค่าให้เป็น `false` ก่อน แล้วค่อยเปลี่ยนกลับตอนเรา setup SSL ในตอนท้าย:
>
> ```ruby
> config.force_ssl = false
> ```
> {: file="config/environments/production.rb"}
{: .prompt-info }

เริ่มแรกให้รัน `deploy:check` ก่อน เพื่อเป็นการสร้างโครงสร้างของโฟลเดอร์บนเครื่องที่เราจะ deploy:

```sh
cap production deploy:check
```
{: file="Local Machine"}

แน่นอนว่ามาถึงตรงนี้จะมี error ว่าไม่มีไฟล์อย่าง `database.yml`, `master.key`, `puma.rb` ซึ่งเป็นรายชื่อไฟล์ที่เรากำหนดใน `linked_files` นั่นแหละ ให้เราทำการ copy ไปไว้ในเครื่อง deploy ก่อน:

```sh
scp config/database.yml [DEPLOY_USER]@[PRODUCTION_IP]:[DEPLOY_TO]/shared/config/database.yml
scp config/master.key [DEPLOY_USER]@[PRODUCTION_IP]:[DEPLOY_TO]/shared/config/master.key
scp config/puma.rb [DEPLOY_USER]@[PRODUCTION_IP]:[DEPLOY_TO]/shared/config/puma.rb
```
{: file="Local Machine"}

หรือจะสร้าง task ไว้ใช้:

```ruby
desc 'Copy linked files'
task :copy_linked_files do
  on roles(:app) do |server|
    user = fetch(:user)
    path = fetch(:deploy_to)
    linked_files = fetch(:linked_files)

    cmd = []
    linked_files.each do |linked_file|
      cmd << "scp #{linked_file} #{user}@#{server.hostname}:#{path}/shared/#{linked_file}"
    end
    exec cmd.join('&&')
  end
end
```
{: file="lib/capistrano/tasks/copy_linked_files.rake"}

เสร็จแล้วเราสามารถรัน task นี้ได้เลย:

```sh
cap production copy_linked_files
```
{: file="Local Machine"}

หลังจากรัน `deploy:check` ก็จะผ่านทั้งหมด

ต่อไปเป็นการติดตั้ง Puma systemd service:

```sh
cap production puma:install
```
{: file="Local Machine"}

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

หลังจากติดตั้งผ่านให้ remote เข้าไปแก้ไข `puma.rb` เสียก่อน โดยไฟล์จะอยู่ที่ `/home/[DEPLOY_TO]/shared/config/puma.rb`:

```ruby
#!/usr/bin/env puma

directory "/home/[DEPLOY_TO]/current"
rackup "/home/[DEPLOY_TO]/current/config.ru"
environment "production"

tag ""

pidfile "/home/[DEPLOY_TO]/shared/tmp/pids/puma.pid"
state_path "/home/[DEPLOY_TO]/shared/tmp/pids/puma.state"
stdout_redirect "/home/[DEPLOY_TO]/shared/log/puma_access.log", "/home/[DEPLOY_TO]/shared/log/puma_error.log", true

threads 0, 16

bind "unix:///home/[DEPLOY_TO]/shared/tmp/sockets/puma.sock"

workers 5

restart_command "bundle exec puma"

prune_bundler

before_restart do
  puts "Refreshing Gemfile"
  ENV["BUNDLE_GEMFILE"] = ""
end
```
{: file="/home/[DEPLOY_TO]/shared/config/puma.rb"}

> `prune_bundler` บอก Puma ให้ start worker โดยไม่โหลด bundler environment ของ master process ใหม่ — ทำให้ restart ไปใช้ Gemfile ใหม่ได้โดยไม่ต้อง stop+start manual ส่วน `before_restart` block ล้างค่า `BUNDLE_GEMFILE` กันสับสนระหว่าง release เก่ากับใหม่ตอน hot restart
{: .prompt-info }

> ให้แทนที่ `[DEPLOY_TO]` ด้วยที่อยู่ของ app ตัวอย่างเช่น `deploy/app_name`
{: .prompt-info }

> ตรวจดู `/home/[DEPLOY_TO]/shared/config/database.yml` เสียก่อน เราจะต้อง config ในส่วนของ production ให้เข้ากับ database ที่เราเลือกใช้ ในที่นี้เราใช้ SQLite ดังนั้นจึงต้องระบุที่ที่จะเก็บไฟล์ `production.sqlite3` ด้วยตัวอย่างในที่นี้เช่น:
>
> ```yaml
> production:
>   <<: *default
>   database: /home/[DEPLOY_TO]/shared/storage/production.sqlite3
> ```
> {: file="/home/[DEPLOY_TO]/shared/config/database.yml"}
{: .prompt-warning }

จากนั้นก็ทำการ deploy เป็นครั้งแรกให้ผ่านก่อน:

```sh
cap production deploy
```
{: file="Local Machine"}

หากเราดูสถานะของ Puma ก็น่าจะ active ด้วย:

```sh
cap production puma:status
```
{: file="Local Machine"}

ต่อไปเราจะสร้างไฟล์ config nginx สำหรับ app ของเราโดยจะสร้างไว้ใน `/etc/nginx/sites-available/` ชื่อไฟล์เป็นอะไรก็ได้ ในที่นี้จะใช้รูปแบบ `[app_name]_puma_production`:

```sh
sudo vi /etc/nginx/sites-available/my_app_puma_production
```
{: file="root@1.2.3.4"}

โดยที่มีการตั้งค่าตามนี้ก่อน:

```nginx
upstream my_app_puma_production {
  server unix:///home/[DEPLOY_TO]/shared/tmp/sockets/puma.sock fail_timeout=0;
}

server {
  listen 80;
  server_name localhost my_app.local;
  root /home/[DEPLOY_TO]/current/public;
  try_files $uri/index.html $uri @my_app_puma_production;

  client_max_body_size 4G;
  keepalive_timeout 10;

  error_page 500 502 504 /500.html;
  error_page 503 @503;

  location @my_app_puma_production {
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $host;
    proxy_redirect off;
    proxy_headers_hash_max_size 1024;
    proxy_headers_hash_bucket_size 128;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header X-Forwarded-Proto http;
    proxy_pass http://my_app_puma_production;
    # limit_req zone=one;
    access_log /home/[DEPLOY_TO]/shared/log/nginx.access.log;
    error_log /home/[DEPLOY_TO]/shared/log/nginx.error.log;
  }

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  location = /50x.html {
    root html;
  }

  location = /404.html {
    root html;
  }

  location @503 {
    error_page 405 = /system/maintenance.html;
    if (-f $document_root/system/maintenance.html) {
      rewrite ^(.*)$ /system/maintenance.html break;
    }
    rewrite ^(.*)$ /503.html break;
  }

  if ($request_method !~ ^(GET|HEAD|PUT|PATCH|POST|DELETE|OPTIONS)$ ){
    return 405;
  }

  if (-f $document_root/system/maintenance.html) {
    return 503;
  }
}
```
{: file="/etc/nginx/sites-available/my_app_puma_production"}

> `upstream` block ประกาศ backend ของ nginx เป็น Puma ที่ฟัง unix socket (เร็วกว่า TCP port เพราะข้าม TCP/IP stack) — ทุก request ที่ nginx จัดการไม่ได้เอง (เช่น `/assets/` ที่ serve static) จะส่งต่อมาที่ socket นี้
{: .prompt-info }

> อย่าลืมแก้ไข `my_app_puma_production` ในไฟล์นี้เป็นชื่อที่ต้องการ แต่ก็แนะนำให้เป็นรูปแบบนี้ `[app_name]_puma_production`
{: .prompt-tip }

เสร็จแล้วก็สร้าง symlink ของการตั้งค่านี้ไปที่ sites-enabled:

```sh
sudo ln -s /etc/nginx/sites-available/my_app_puma_production /etc/nginx/sites-enabled/my_app_puma_production
```
{: file="root@1.2.3.4"}

จากนั้นรันเทสของ nginx ดูว่าไม่มีปัญหาอะไร:

```sh
sudo nginx -t
```
{: file="root@1.2.3.4"}

แล้วทำการ reload, restart สักครั้ง:

```sh
sudo service nginx reload
sudo service nginx restart
```
{: file="root@1.2.3.4"}

แล้วไปที่ browser แล้วไปที่ PRODUCTION_IP:


![](https://i.imgur.com/WjQdXBX.png)


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

มาถึงตรงนี้เราก็พร้อมที่จะเอา domain ชี้มาที่ server เราได้แล้ว ถ้าทำเรียบร้อยแล้ว ก็มาเริ่มในขั้นต่อไปกันเลย

เริ่มจากแก้ไข nginx config ก่อน:

```nginx
upstream my_app_puma_production {
  server unix:///home/[DEPLOY_TO]/shared/tmp/sockets/puma.sock fail_timeout=0;
}

server {
  server_name mydomain.com www.mydomain.com;
  root /home/[DEPLOY_TO]/current/public;

  access_log /home/[DEPLOY_TO]/current/log/nginx.access.log;
  error_log /home/[DEPLOY_TO]/current/log/nginx.error.log info;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  try_files $uri/index.html $uri @my_app_puma_production;

  location @my_app_puma_production {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Ssl on; # Optional
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header X-Forwarded-Host $host;

    proxy_redirect off;

    proxy_pass http://my_app_puma_production;
  }

  location /cable {
    proxy_pass http://my_app_puma_production;
    proxy_http_version 1.1;
    proxy_set_header Upgrade "websocket";
    proxy_set_header Connection "Upgrade";
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_redirect off;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 100M;
  keepalive_timeout 10;
}
```
{: file="/etc/nginx/sites-enabled/my_app_puma_production"}

ที่สำคัญก็คือแก้ `server_name` เป็น domain name ที่ต้องการ

จากนั้นก็แก้ไข `config.force_ssl` กลับไปเป็น `true` เหมือนเดิม:

```ruby
config.force_ssl = true
```
{: file="config/environments/production.rb"}

ต่อไปก็เป็นการ Setup Let's Encrypt:

```sh
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx
```
{: file="root@1.2.3.4"}

ทำตามขั้นตอนที่ขึ้นมา เลือก domain ที่เราต้องการให้เป็น https

> Let's Encrypt cert มีอายุ 90 วัน — certbot package ของ Ubuntu จะติดตั้ง `certbot.timer` (systemd timer) มาให้ auto-renew อยู่แล้ว ตรวจด้วย `systemctl list-timers | grep certbot` และทดสอบ renew (dry run) ด้วย `sudo certbot renew --dry-run`
{: .prompt-tip }

จากนั้นก็ให้เรา commit และ push สิ่งที่เราแก้ไขขึ้น GitHub ให้เรียบร้อยจากนั้นก็ทำการ deploy อีกครั้ง:

```sh
cap production deploy
```
{: file="Local Machine"}


## Conclusion

ยังคงเป็นการ deploy แบบง่าย ๆ อยู่ ยังคงต้องทำการติดตั้ง database และ background process กันต่อ — ดูที่ [How to install MySQL 5.7 on Ubuntu 22.04](/posts/how-to-install-mysql-5-7-on-ubuntu-22-04/) สำหรับ database และ [Deploy Solid Queue with Capistrano](/posts/deploy-solid-queue-with-capistrano/) สำหรับ background job

ถ้าไม่อยากดูแล Nginx/Puma/Capistrano เองและพร้อมรับ Docker เป็น dependency ลองดู [Deploy Rails apps to Ubuntu servers with Kamal](/posts/deploy-rails-apps-to-ubuntu-servers-with-kamal/) ที่ใช้ workflow ผ่าน container แทน

## References

- [Capistrano documentation](https://capistranorb.com/)
- [capistrano3-puma](https://github.com/seuros/capistrano-puma)
- [capistrano-rails](https://github.com/capistrano/rails)
- [capistrano-rbenv](https://github.com/capistrano/rbenv)
- [rbenv-vars plugin](https://github.com/rbenv/rbenv-vars)
- [Puma documentation](https://puma.io/)
- [Nginx + Puma deployment guide (Ruby on Rails Guides)](https://guides.rubyonrails.org/configuring.html#configuring-puma)
- [Let's Encrypt with Certbot](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal)
