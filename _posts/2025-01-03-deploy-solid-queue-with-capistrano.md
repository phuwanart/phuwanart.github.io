---
layout: post
title: Deploy Solid Queue with Capistrano
categories: Posts
tags:
- rails
- deploy
- capistrano
date: 2025-01-03 11:52 +0700
---
```
/home/<deploy user>/.config/systemd/user/solid_queue.service
```
{:file='Remote Server'}

```
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
{:file='config/deploy.rb'}

## References

- https://world.hey.com/robzolkos/how-i-deploy-solid-queue-with-capistrano-487b4a31