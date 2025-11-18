---
layout: post
title: Deploy Rails 8.1 with Kamal on a VM (Experimental)
categories: Posts
tags:
- rails
- kamal
- docker
- deploy
date: 2025-11-05 13:55 +0700
---

หลังจากที่ [rails 8.1](https://rubyonrails.org/blog/2023/04/03/rails-8-1-released.html) ปล่อยออกมา มีอย่างหนึ่งที่เห็นแล้วเป็นอะไรที่กำลังรออยู่เลยก็คือ **Registry-Free Kamal Deployments** ที่จะทำให้เราสามารถ deploy rails application ได้โดยไม่ต้องใช้ registry ที่เป็น online อย่างพวก [docker hub](https://hub.docker.com/) อีกแล้ว เพราะมันได้ free private registry แค่ 1 ตัวเท่านั้น

## Getting Started

ในการทดลองนี้จะลอง deploy บน vm ก่อน ซึ่งน่าจะเป็นแนวทางในการ deploy บน server จริง ๆ ของเรา หรือบน cloud service ที่มีการจัดการ vps ให้เรา อย่างเช่น [DigitalOcean](https://www.digitalocean.com/)

### Create a Server

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
{:file='root@1.2.3.4'}

### Create Deploy User

```sh
sudo adduser deploy
```
{:file='root@1.2.3.4'}

และเพิ่ม user `deploy` ไปยัง `sudo` group

```sh
sudo adduser deploy sudo
```
{:file='root@1.2.3.4'}

จากนั้นก็ login เข้า deploy user

```sh
su deploy
```
{:file='root@1.2.3.4'}

เพิ่ม group `docker` ซึ่งเป็นส่วนที่ kamal จะใช้ในการ deploy

```sh
sudo addgroup docker
sudo usermod -aG docker $USER
```
{:file='deploy@1.2.3.4'}

> ต้อง logout แล้ว login ใหม่ (เพื่อให้สิทธิ์กลุ่ม docker มีผล)
{: .prompt-tip }

แล้วสร้าง directory สำหรับ storage สำหรับ rails application

```sh
mkdir blog_storage
```
{:file='deploy@1.2.3.4'}

จากนั้นแก้ไฟล์ sudoers

```sh
sudo visudo
```
{:file='deploy@1.2.3.4'}

จากนั้นเพิ่ม `deploy  ALL=(ALL) NOPASSWD:ALL` หลังบรรทัด `%sudo   ALL=(ALL:ALL) ALL`

```sh
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

deploy  ALL=(ALL) NOPASSWD:ALL
```
{:file='deploy@1.2.3.4'}

### Add SSH Key for Deploy

#### Using `ssh-copy-id`

```sh
brew install ssh-copy-id
```
{:file="Local Machine"}

```sh
ssh-copy-id deploy@1.2.3.4
```
{:file="Local Machine"}

```sh
ssh deploy@1.2.3.4
```
{:file="Local Machine"}

#### Using `authorized-key`

```sh
# for RSA SSH key
cat ~/.ssh/id_rsa.pub
```
{:file="Local Machine"}

```sh
# for ED25519 SSH key
cat ~/.ssh/id_ed25519.pub
```
{:file="Local Machine"}

```sh
mkdir ~/.ssh && touch ~/.ssh/authorized_keys
```
{:file='deploy@1.2.3.4'}

```sh
sudo vi ~/.ssh/authorized_keys
```
{:file='deploy@1.2.3.4'}

```sh
ssh-ed25519 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX user@computer
```
{:file="~/.ssh/authorized_keys"}

```sh
ssh deploy@1.2.3.4
```
{:file="Local Machine"}

### Install Docker Desktop or OrbStack

```sh
brew install --cask docker-desktop
```
{:file="Local Machine"}

หรือ

```sh
brew install --cask orbstack
```
{:file="Local Machine"}

### Create Rails Application

```sh
rails new blog
cd blog
rails generate controller home index
```
{:file='Local Machine'}

ตั้ง root ใช้เรียบร้อย

```ruby
root 'home#index'
```
{:file="config/routes.rb"}

### Configuring Kamal

ไปตรวจดู `config/deploy.yml` แก้ไขตามนี้

```diff
# Name of your application. Used to uniquely configure containers.
service: blog

# Name of the container image (use your-user/app-name on external registries).
image: blog

# Deploy to these servers.
servers:
  web:
-   - 192.168.0.1
+   - [ip_address ของ server]
  # job:
  #   hosts:
  #     - 192.168.0.1
  #   cmd: bin/jobs

# Enable SSL auto certification via Let's Encrypt and allow for multiple apps on a single web server.
# If used with Cloudflare, set encryption mode in SSL/TLS setting to "Full" to enable CF-to-app encryption.
#
# Using an SSL proxy like this requires turning on config.assume_ssl and config.force_ssl in production.rb!
#
# Don't use this when deploying to multiple web servers (then you have to terminate SSL at your load balancer).
#
# proxy:
#   ssl: true
#   host: app.example.com

# Where you keep your container images.
registry:
  # Alternatives: hub.docker.com / registry.digitalocean.com / ghcr.io / ...
  server: localhost:5555

  # Needed for authenticated registries.
  # username: your-user

  # Always use an access token rather than real password when possible.
  # password:
  #   - KAMAL_REGISTRY_PASSWORD

# Inject ENV variables into containers (secrets come from .kamal/secrets).
env:
  secret:
    - RAILS_MASTER_KEY
  clear:
    # Run the Solid Queue Supervisor inside the web server's Puma process to do jobs.
    # When you start using multiple servers, you should split out job processing to a dedicated machine.
    SOLID_QUEUE_IN_PUMA: true

    # Set number of processes dedicated to Solid Queue (default: 1)
    # JOB_CONCURRENCY: 3

    # Set number of cores available to the application on each server (default: 1).
    # WEB_CONCURRENCY: 2

    # Match this to any external database server to configure Active Record correctly
    # Use blog-db for a db accessory server on same machine via local kamal docker network.
    # DB_HOST: 192.168.0.2

    # Log everything from Rails
    # RAILS_LOG_LEVEL: debug

# Aliases are triggered with "bin/kamal <alias>". You can overwrite arguments on invocation:
# "bin/kamal logs -r job" will tail logs from the first server in the job section.
aliases:
  console: app exec --interactive --reuse "bin/rails console"
  shell: app exec --interactive --reuse "bash"
  logs: app logs -f
  dbc: app exec --interactive --reuse "bin/rails dbconsole --include-password"

# Use a persistent storage volume for sqlite database files and local Active Storage files.
# Recommended to change this to a mounted volume path that is backed up off server.
volumes:
- - "blog_storage:/rails/storage"
+ - "./blog_storage:/rails/storage"

# Bridge fingerprinted assets, like JS and CSS, between versions to avoid
# hitting 404 on in-flight requests. Combines all files from new and old
# version inside the asset_path.
asset_path: /rails/public/assets


# Configure the image builder.
builder:
  arch: amd64

  # # Build image via remote server (useful for faster amd64 builds on arm64 computers)
  # remote: ssh://docker@docker-builder-server
  #
  # # Pass arguments and secrets to the Docker build process
  # args:
  #   RUBY_VERSION: ruby-3.4.7
  # secrets:
  #   - GITHUB_TOKEN
  #   - RAILS_MASTER_KEY

# Use a different ssh user than root
-# ssh:
-#   user: app
+ssh:
+  user: deploy

# Use accessory services (secrets come from .kamal/secrets).
# accessories:
#   db:
#     image: mysql:8.0
#     host: 192.168.0.2
#     # Change to 3306 to expose port to the world instead of just local network.
#     port: "127.0.0.1:3306:3306"
#     env:
#       clear:
#         MYSQL_ROOT_HOST: '%'
#       secret:
#         - MYSQL_ROOT_PASSWORD
#     files:
#       - config/mysql/production.cnf:/etc/mysql/my.cnf
#       - db/production.sql:/docker-entrypoint-initdb.d/setup.sql
#     directories:
#       - data:/var/lib/mysql
#   redis:
#     image: valkey/valkey:8
#     host: 192.168.0.2
#     port: 6379
#     directories:
#       - data:/data
```
{:file="config/deploy.yml"}

จากนั้น commit ให้เรียบร้อยก่อนจะเริ่ม deploy กัน

รันคำสั่ง `kamal setup`

หลังจาก setup เสร็จก็เปิดเว็บเบราว์เซอร์ไปที่ ip_address ของ server ที่เรา deploy ได้เลย

และครั้งต่อไปเราจะใช้ `kamal deploy` เพื่อ deploy ใหม่

## Conclusion

ในบทความนี้เราได้ทำการ deploy rails 8.1 บน server ด้วย kamal ได้สำเร็จแบบ minimal setup แล้ว

## References

- [Kamal Deployment: The Newest Form of Self-Torture](https://alec-c4.com/posts/2025-04-02-kamal/)
- [Rails Application Deployment on Digital Ocean](https://medium.com/@qasimali7566675/rails-application-deployment-on-digital-ocean-8483d3f10813)
- [Ultimate Guide to Server Hardening for Kamal](https://blog.cloud66.com/ultimate-guide-to-server-hardening-for-kamal)
