---
layout: post
title: Rails 7 Deployment with Ubuntu, Capistrano, Puma and Nginx
categories: Posts
tags: rails deploy ubuntu capistrano
---

ในที่นี้เราจะทำการ deploy rails ไปที่ ubuntu server กัน โดยเราจะใช้ stack ดังนี้:

- Ubuntu 22.04 LTS
- Ruby 3.2.2
- Rails 7.0.8
- Puma 5
- Capistrano

## Setup Ubuntu

หลังจากติดตั้ง ubuntu server เรียบร้อยแล้วเราจะสามารถ login เป็น root (หรือชื่ออะไรก็ตามที่ใส่ตอนลง ubuntu) เข้า server ได้ด้วยคำสั่งนี้ โดยเปลี่ยน `1.2.3.4` เป็น IP ของเครื่อง

```sh
ssh root@1.2.3.4
```
{:file='Local Machine'}

พอ login เข้าได้แล้วก็ทำการอัพเดทระบบสักนึดนึง

```sh
sudo apt update && sudo apt upgrade -y && sudo apt-get autoremove && sudo reboot
```
{:file='root@1.2.3.4'}

ติดตั้ง `net-tools` เพื่อดู IP ของ server

```sh
sudo apt install net-tools
```
{:file='root@1.2.3.4'}

ติดตั้ง `vim` เป็น text editor (แล้วแต่ถนัด)

```sh
sudo apt install vim
```
{:file='root@1.2.3.4'}

### Creating a Deploy User

สร้าง deploy user เพื่อใช้สำหรับลง เราจะไม่ใช้ root เพื่อความปลอดภัย

login เข้า root แล้วสร้าง user deploy และเพิ่มเข้า group sudo

```sh
sudo adduser deploy
sudo adduser deploy sudo
exit
```
{:file='root@1.2.3.4'}

จากนั้นเราจะเพิ่ม ssh key ของเรื่องเราไปที่ server เพื่อจะได้ login server ได้ง่าย ๆ โดยเครื่องมือที่เราจะใช้ชื่อ `ssh-copy-id` ถ้าเราใช้ Mac เราสามารถ `brew install ssh-copy-id` ได้เลย

```sh
ssh-copy-id root@1.2.3.4
ssh-copy-id deploy@1.2.3.4
```
{:file='Local Machine'}

หลังจากนี้เราจะ login เข้า deploy เพื่อทำขั้นตอนต่อไป

## Installing Ruby

เริ่มจากลง package ที่จำเป็นสำหรับการ compile ruby เสียก่อน

```sh
sudo apt install -y git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates
```
{:file='deploy@1.2.3.4'}

ในที่นี้เราจะใช้ rbenv เป็นตัวจัดการเวอร์ชั่นของ ruby

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
{:file='deploy@1.2.3.4'}

เสร็จแล้วก็ตรวจสอบสักนิดว่า rbenv ใช้งานได้แล้ว

```sh
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-doctor | bash
```
{:file='deploy@1.2.3.4'}

ทำการอัพเดท gem ของระบบ

```sh
gem update --system
```
{:file='deploy@1.2.3.4'}

และ

```sh
gem install bundler
```
{:file='deploy@1.2.3.4'}

## Installing NGINX

เราจะใช้ nginx เป็น webserver สามารถติดตั้งได้ตามขั้นตอนปกติ

```sh
sudo apt install nginx
```
{:file='deploy@1.2.3.4'}

หลังจากนั้นเราจะลบ default ที่ติดมาตอนลงออก

```sh
sudo rm /etc/nginx/sites-enabled/default
```
{:file='deploy@1.2.3.4'}

และทำการ restart nginx

```sh
sudo service nginx restart
```
{:file='deploy@1.2.3.4'}


## Installing Redis

```sh
sudo apt install redis
sudo service redis-server status
```
{:file='deploy@1.2.3.4'}

## Create Rails App

## Setup SSH keys

ในการ deploy เราจะต้องมีที่เก็บ code ไว้ ในที่นี้เราจะใช้ github ก่อนอื่นเราต้องเช็คก่อนเราสามารถต่อกับ github ได้

```sh
ssh -T git@github.com
```
{:file='deploy@1.2.3.4'}

ก่อนอื่นสร้าง ssh-key

```sh
ssh-keygen -t rsa
```
{:file='deploy@1.2.3.4'}

ให้เว้นว่างไม่ต้องใส่ password

แล้วเราสามารถดู key ที่สร้างด้วยคำสั่งนี้

```sh
cat ~/.ssh/id_rsa.pub
```
{:file='deploy@1.2.3.4'}

ทำตามขั้นตอนนี้ [Set up deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#set-up-deploy-keys) จากนั้นรันคำสั่ง `ssh -T git@github.com` อีกครั้ง ต้องขึ้นว่าสำเร็จ เป็นอันจบ

## Setup Capistrano

```ruby
group :development do
  gem "bcrypt_pbkdf"
  gem "capistrano"
  gem "capistrano3-puma", "~> 5"
  gem "capistrano-rails"
  gem "capistrano-rbenv"
  gem "capistrano-rbenv-vars"
  gem "ed25519"
end
```
{:file='Gemfile'}

```sh
bundle
```
{:file='Local Machine'}

```sh
cap install
```
{:file='Local Machine'}

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
require "capistrano/rbenv_vars"
# require "capistrano/chruby"
require "capistrano/bundler"
require "capistrano/rails"
require "capistrano/rails/assets"
require "capistrano/rails/migrations"
# require "capistrano/passenger"
require "capistrano/puma"
install_plugin Capistrano::Puma
install_plugin Capistrano::Puma::Systemd
require "capistrano/puma/nginx"
install_plugin Capistrano::Puma::Nginx

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```
{:file='Capfile'}

```ruby
# config valid for current version and patch releases of Capistrano
lock "~> 3.18.0"

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
append :linked_files, "config/database.yml", "config/master.key"

# Default value for linked_dirs is []
# append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "vendor", "storage"
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "vendor/bundle", "storage"

# Default value for default_env is {}
# set :default_env, { path: "/opt/ruby/bin:$PATH" }

# Default value for local_user is ENV['USER']
# set :local_user, -> { `git config user.name`.chomp }

# Default value for keep_releases is 5
set :keep_releases, 5

# Uncomment the following to require manually verifying the host key before first deploy.
# set :ssh_options, verify_host_key: :secure

set :rbenv_ruby, "3.2.2"

set :puma_workers, 5
```
{:file='config/deploy.rb'}

```ruby
server "[PRODUCTION_IP]", user: "[DEPLOY_USER]", roles: %w[app db web]
```
{:file='config/deploy/production.rb'}

## Setup Production Variables

## Deploy

```sh
cap production deploy:check
```
{:file='Local Machine'}

```sh
scp config/database.yml femto@192.192.1.184:/home/femto/whowedding/shared/config/database.yml
scp config/master.key femto@192.192.1.184:/home/femto/whowedding/shared/config/master.key
```
{:file='Local Machine'}


```sh
cap production puma:config
```
{:file='Local Machine'}

```sh
cap production puma:systemd:config
cap production puma:systemd:enable
```
{:file='Local Machine'}

> ในขั้นตอนนี้อาจจะมีการขอสิทธิ์ sudo ในตอนรันคำสั่ง ซึ่งจะมี error ประมาณนี้:
> ```sh
> 01 sudo /bin/systemctl restart puma_whowedding_production
> 01 sudo
> 01 :
> 01 a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
> 01
> 01 sudo
> 01 :
> 01 a password is required
> 01
> ```
> {:file='Local Machine'}
>
> ก็ให้เรา login เข้า deploy แล้วไปแก้ไขได้ดังนี้:
>
> ```sh
> sudo visudo
> ```
> {:file='deploy@1.2.3.4'}
> 
> ```sh
> # Allow members of group sudo to execute any command
> %sudo   ALL=(ALL:ALL) ALL
> 
> femto   ALL=(ALL) NOPASSWD:ALL
> ```
> {:file='deploy@1.2.3.4'}
{: .prompt-danger }

```sh
cap production puma:nginx_config
```
{:file='Local Machine'}

```sh
sudo service nginx restart
```
{:file='deploy@1.2.3.4'}

```sh
cap production deploy
```
{:file='Local Machine'}

> ```sh
> tail -f /var/log/nginx/error.log
>
> ...failed (13: Permission denied)...
> ```
> {:file='deploy@1.2.3.4'}
>
> ```sh
> cd /home
> sudo chmod o=rx ubuntu/
> ```
> {:file='deploy@1.2.3.4'}
{: .prompt-danger }

> ```sh
> Permission denied @ rb_io_reopen - /home/ubuntu/whowedding/shared/log/puma_access.log (Errno::EACCES)
> ```
> {:file='deploy@1.2.3.4'}
>
> ```sh
> sudo chown ubuntu:ubuntu -R /home/ubuntu/whowedding/
> ```
> {:file='deploy@1.2.3.4'}
{: .prompt-danger }

```sh
cap production puma:restart
```
{:file='Local Machine'}

## References

- [https://gorails.com/deploy/ubuntu/22.04](https://gorails.com/deploy/ubuntu/22.04)
- [https://dev.to/1klap/rails-7-production-deploy-from-scratch-ubuntu-2204-edition-pm7](https://dev.to/1klap/rails-7-production-deploy-from-scratch-ubuntu-2204-edition-pm7)
- [https://www.matthewhoelter.com/2020/11/10/deploying-ruby-on-rails-for-ubuntu-2004.html](https://www.matthewhoelter.com/2020/11/10/deploying-ruby-on-rails-for-ubuntu-2004.html)