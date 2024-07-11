---
layout: post
title: How to deploy Rails 7.2 from scratch with Capistrano, Puma and Nginx on Ubuntu 24.04
categories: Posts
tags: rails ubuntu deploy nginx puma capistrano
---

ครั้งก่อนได้เขียนเรื่องนี้ไปรอบหนึ่งแล้ว แต่หลังจากที่ rails ออกเวอร์ชั่น 7.1 ก็มีอะไรเปลี่ยนแปลงไปนิดหน่อย ประกอบกับตอนนี้ 7.2 ก็ออกตัวเบต้าแล้ว และมี ubuntu ที่ออกเวอร์ชั่น 24.04 ด้วย แต่หลังจากที่ไปลองเอาที่เขียนก่อนหน้าไปทำการลงดูก็พบว่ามีอะไรเปลี่ยนไปอยู่เหมือนกัน เลยคิดว่าเอามาเขียนอีกรอบดีกว่า

ในหัวข้อนี้จะประกอบไปด้วย tech stack ดังนี้:

- Ubuntu 24.04 LTS (VirtualBox)
- Ruby 3.3.4
- Rails 7.2.0.beta2
- Capistrano 3.19.1
- Puma 6
- Nginx

## Configure Production Server

หลังจากติดตั้ง ubuntu เสร็จ ถ้าเป็นเวอร์ชั่น 22.04 นั้นเราสามารถที่จะ ssh เข้าไปได้เลย แต่พอมาเป็น 24.04 กลับทำไม่ได้ ซึ่งเราจะต้องตั้งค่า ssh เสียก่อน:[^setup-ssh]

[^setup-ssh]: [Setting Up and Securing SSH on Ubuntu 22.04: A Comprehensive Guide](https://serverastra.com/docs/Tutorials/Setting-Up-and-Securing-SSH-on-Ubuntu-22.04%253A-A-Comprehensive-Guide)

```sh
sudo apt install openssh-server
sudo ufw status
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo systemctl restart ssh
sudo systemctl status ssh
```
{:file='Remote Server'}

หลังจากเสร็จแล้วเราจะใช้เครื่อง local ในการ remote เข้ามาที่ server ให้เหมือนสภาพจริง ๆ ที่เราจะทำงานกัน

ให้เราติดตั้ง `net-tools` ซะก่อน เพื่อจะได้ดู IP ของ server ได้จาก `ifconfig`:

```sh
sudo apt install net-tools
ifconfig
```
{:file='Remote Server'}

สมมุติว่า IP ที่ได้คือ `1.2.3.4` นะ

พอได้ IP มาแล้วให้ไปใช้เครื่อง local ของเราให้เรา ssh เข้าไปที่ server โดยใช้ขื่อ root หรือชื่ออะไรก็ตามที่กรอกตอนติดตั้ง ubuntu:

```sh
ssh root@1.2.3.4 
```
{:file='Local Machine'}

พอ login ได้แล้ว เราจะทำการอัพเดทระบบกันก่อน:

```sh
sudo apt update && sudo apt upgrade -y && sudo apt-get autoremove && sudo reboot
```
{:file='root@1.2.3.4'}

และทำการติดตั้ง packages ที่จำเป็น:

```sh
sudo apt install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates
```
{:file='root@1.2.3.4'}

## Create Deploy User

เราจะสร้าง user บน server เพื่อใช้ deploy rails app ของเรา ในที่นี้ให้ชื่อ `deploy` ซึ่งในความคิดผมนั้น อาจจะเอาชื่อ project ชื่อ app มาใช้เป็นชื่อ user ก็ได้ เพราะว่าใน server ของเรา อาจจะมีหลาย ๆ ตัว deploy ไว้ด้วยกัน เพื่อให้ง่ายต่อการจัดการ

```sh
sudo adduser deploy
sudo adduser deploy sudo
exit
```
{:file='root@1.2.3.4'}

ต่อไปเราจะเพิ่ม ssh key ไปที่ server เพื่อให้สามารถ login ได้อย่างรวดเร็วโดยจะใช้เครื่องมือที่เรียกว่า `ssh-copy-id`

หากใช้เครื่อง Mac จะต้องติดตั้ง `ssh-copy-id` ด้วย homebrew เสียก่อน `brew install ssh-copy-id`:

```sh
ssh-copy-id root@1.2.3.4
ssh-copy-id deploy@1.2.3.4
```
{:file='Local Machine'}

ตอนนี้เราสามารถที่จะเข้า root และ deploy ได้โดยไม่ต้องใส่ password แล้ว:

```sh
ssh deploy@1.2.3.4
```
{:file='Local Machine'}

แต่ถ้า `ssh-key is not recognized` ขึ้นมาให้รันคำสั่งตามนี้:

```sh
eval $(ssh-agent)  
ssh-add
```
{:file='Local Machine'}

## Install Ruby

เราจะใช้ `rbenv` ในการติดตั้ง `ruby` โดยทำตามนี้:

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
{:file='deploy@1.2.3.4'}

ตรวจสอบว่าติดตั้งเรียบร้อยหรือไม่ด้วย`rbenv-doctor`:

```sh
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-doctor | bash
```
{:file='deploy@1.2.3.4'}

ตรวจดูว่า `ruby` ที่ลงไปถูกเวร์ชั่นมั้ย:

```sh
ruby -v
```
{:file='deploy@1.2.3.4'}

ตั้งค่า `.gemrc` เพื่อให้ตอนที่ลอง gem จะได้ไม่ต้องลง document มาด้วย (เพื่อว่าความไว):

```sh
echo "gem: --no-document" > ~/.gemrc
```
{:file='deploy@1.2.3.4'}

ติดตั้ง `bundler`:

```sh
gem install bundler
```
{:file='deploy@1.2.3.4'}

หลังจากติดตั้งอาจจะบอกให้อัพเดท:

```sh
gem update --system
```
{:file='deploy@1.2.3.4'}

ดูว่า `bundler` ที่ติดตั้งถูกต้องมั้ย:

```sh
bundle -v
```
{:file='deploy@1.2.3.4'}

## Install NGINX

ทำการติดตั้ง `nginx` ตามนี้:

```sh
sudo apt install nginx
```
{:file='deploy@1.2.3.4'}

และเปิด browser ตาม IP:

```sh
open http://1.2.3.4
```
{:file='Local Machine'}

เจอข้อความต้อนรับก็เป็นอันเรียบร้อย ซึ่งตอนนี้เราจะยังไม่ทำอะไรกับมันมาก ไปต่อกันเลย:

![](https://i.imgur.com/MY8Jl29.png)



## Install Redis

ติดตั้ง redis เพราะว่า rails อาจจะต้องการในบางฟีเจอร์ สามารถทำตามขั้นตอนนี้ได้เลย[^redis]:

[^redis]: [How to Install Redis on Ubuntu/Debian and Ensure Auto-Restart on System Reboot](https://medium.com/@kbablu557/how-to-install-redis-on-ubuntu-debian-and-ensure-auto-restart-on-system-reboot-4c9fe675bdb1)

```sh
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt update
sudo apt install redis
```
{:file='deploy@1.2.3.4'}

ดูเวอร์ชั่น:

```sh
$ redis-cli --version
redis-cli 7.2.5
```
{:file='deploy@1.2.3.4'}

ทดสอบว่าทำงานได้:

```sh
$ redis-cli ping
PONG
```
{:file='deploy@1.2.3.4'}

จากนั้นตั้งค่าให้ทำงานตอนเครื่องบูทด้วย:

```sh
sudo systemctl enable redis-server
```
{:file='deploy@1.2.3.4'}

## Create Rails App

เราจะสร้าง app แบบง่าย ๆ เพื่อเป็นการทดสอบการ deploy โดยเป็น rails 7.2.0.beta2 หากว่ายังไม่ได้ติดตั้งให้ติดตั้งด้วย `gem install rails --pre`:

```sh
rails new my_app
```
{:file='Local Machine'}

จากนั้นก็เอา app นี้ขึ้น github ให้เรียบร้อย

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
{:file='deploy@1.2.3.4'}

เริ่มจากสร้าง ssh key ใหม่:

```sh
ssh-keygen -t rsa
```
{:file='deploy@1.2.3.4'}

เมื่อทำตามนี้มาถึงตอนที่ถามพาสเวิร์ดก็ให้ปล่อยว่างไว้ แล้ว SSH key จะถูกสร้างไว้ที่ `~/.ssh/id_rsa.pub` ซึ่งสามารถเข้าไปดูได้ด้วยทำสั่ง:

```sh
cat ~/.ssh/id_rsa.pub
```
{:file='deploy@1.2.3.4'}

ไปที่ Settings ของ repository บน GitHub แล้วไปที่เมนู Deploy keys จากนั้นเอา SSH key ที่ได้ใส่ในช่อง Key ได้เลย:

![](https://i.imgur.com/q9XQRQU.png)


ตอนนี้หาเรารันคำสั่ง `ssh -T git@github.com` ก็จะสำเร็จแล้ว

## Setup Capistrano

เราจะใช้ capistrano สำหรับ automate deployments เพื่อให้การ deploy ทำได้ง่าย ๆ

เพิ่ม gem ที่จำเป็นใน `Gemfile`:

```ruby
group :development do
  gem "bcrypt_pbkdf"
  gem "capistrano"
  gem "capistrano3-puma", "~> 6.beta.1"
  gem "capistrano-rails"
  gem "capistrano-rbenv"
  gem "ed25519"
end
```
{:file='Gemfile'}

ติดตั้ง gem ที่เพิ่มเข้ามา:

```sh
bundle install
```
{:file='Local Machine'}

ใช้ `capistrano` สร้างไฟล์ที่จำเป็นในการ deploy:

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
{:file='Local Machine'}

จากนั้นแก้ไฟล์ `Capfile` ตามนี้:

```ruby
# Load DSL and set up stages
require "capistrano/setup"

# Include default deployment tasks
require "capistrano/deploy"

# Load the SCM plugin appropriate to your project:
#
# require "capistrano/scm/hg"
# install_plugin Capistrano::SCM::Hg
# or
# require "capistrano/scm/svn"
# install_plugin Capistrano::SCM::Svn
# or
require "capistrano/scm/git"
install_plugin Capistrano::SCM::Git

# Include tasks from other gems included in your Gemfile
#
# For documentation on these, see for example:
#
#   https://github.com/capistrano/rvm
#   https://github.com/capistrano/rbenv
#   https://github.com/capistrano/chruby
#   https://github.com/capistrano/bundler
#   https://github.com/capistrano/rails
#   https://github.com/capistrano/passenger
#
# require "capistrano/rvm"
require "capistrano/rbenv"
# require "capistrano/chruby"
require "capistrano/bundler"
require "capistrano/rails/assets"
require "capistrano/rails/migrations"
# require "capistrano/passenger"

require 'capistrano/puma'
install_plugin Capistrano::Puma
install_plugin Capistrano::Puma::Systemd

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```

จากนั้นแก้ไข้ไฟล์ `config/deploy.rb` ตามนี้:

```ruby
# config valid for current version and patch releases of Capistrano
lock "~> 3.19.1"

set :application, "[APP_NAME]"
set :user, "[DEPLOY_USER]"
set :repo_url, "[GIT SSH ADDRESS: git@github.com:username/appname.git]"

# Default branch is :master
# ask :branch, `git rev-parse --abbrev-ref HEAD`.chomp
set :branch, "main"

# Default deploy_to directory is /var/www/my_app_name
# set :deploy_to, "/var/www/my_app_name"
set :deploy_to, "/home/#{fetch :user}/#{fetch :application}"

# Default value for :format is :airbrussh.
# set :format, :airbrussh

# You can configure the Airbrussh format using :format_options.
# These are the defaults.
# set :format_options, command_output: true, log_file: "log/capistrano.log", color: :auto, truncate: :auto

# Default value for :pty is false
# set :pty, true

# Default value for :linked_files is []
append :linked_files, "config/database.yml", "config/master.key", "config/puma.rb"

# Default value for linked_dirs is []
# append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "vendor", "storage"
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "vendor/bundle", ".bundle", "storage"

# Default value for default_env is {}
# set :default_env, { path: "/opt/ruby/bin:$PATH" }

# Default value for local_user is ENV['USER']
# set :local_user, -> { `git config user.name`.chomp }

# Default value for keep_releases is 5
# set :keep_releases, 5

# Uncomment the following to require manually verifying the host key before first deploy.
# set :ssh_options, verify_host_key: :secure

set :rbenv_ruby, "3.3.4"
```

## Setup Production Variables

ก่อนที่จะ deploy ครั้งแรกให้เราเข้าไปสร้างโฟลเดอร์ตาม location ที่กำหนดตาม `set :deploy_to` ใน `config/deploy.rb` จากนั้นสร้างไฟล์ `.rbenv-vars`:

```sh
$ mkdir [DEPLOY_TO]
$ cd [DEPLOY_TO]
$ vi .rbenv-vars
```
{:file='deploy@1.2.3.4'}

จากนั้นเพิ่มตามนี้ลงไนไฟล์:

```
SECRET_KEY_BASE=<random sequence> # use local `rails secret`
RAILS_ENV=production
```
{:file='.rbenv-vars'}

`SECRET_KEY_BASE` เราสามารถสร้างได้ด้วยคำสั่งนี้บนเครื่อง local:

```sh
rails secret
```
{:file='Local Machine'}

## References
