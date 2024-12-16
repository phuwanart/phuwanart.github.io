---
layout: post
title: Rails Engine with Vite
categories: Posts
tags: [rails, rails engine, vite]
---

## Create engine

```sh
rails plugin new my_engine --mountable
```

## Vite configuration

```ruby
spec.add_dependency 'vite_rails'
spec.add_dependency 'vite_ruby'
```
{:file='my_engine.gemspec'}

```diff
require "bundler/setup"

+ require 'vite_ruby'
+ ViteRuby.install_tasks
+ ViteRuby.config.root # Ensure the engine is set as the root.

APP_RAKEFILE = File.expand_path("test/dummy/Rakefile", __dir__)
load "rails/tasks/engine.rake"

load "rails/tasks/statistics.rake"

require "bundler/gem_tasks"
```
{:file='Rakefile'}

```sh
bundle exec vite install
```

```ruby
require "vite_ruby"

module MyEngine
  class Engine < ::Rails::Engine
    isolate_namespace MyEngine

    delegate :vite_ruby, to: :class

    def self.vite_ruby
      @vite_ruby ||= ::ViteRuby.new(root: root, mode: Rails.env)
    end

    # Serves the engine's vite-ruby when requested
    initializer "my_engine.vite_rails.static" do |app|
      if Rails.application.config.public_file_server.enabled
        # this is the right setup when the main application is already
        # using Vite for the theme assets.
        app.middleware.insert_after ActionDispatch::Static,
                                    Rack::Static,
                                    urls: [ "/#{vite_ruby.config.public_output_dir}" ],
                                    root: root.join(vite_ruby.config.public_dir),
                                    header_rules: [
                                      # rubocop:disable Style/StringHashKeys
                                      [ :all, { "Access-Control-Allow-Origin" => "*" } ]
                                      # rubocop:enable Style/StringHashKeys
                                    ]
      else
        # mostly when running the application in production behind NGINX or APACHE
        app.middleware.insert_before 0,
                                     Rack::Static,
                                     urls: [ "/#{vite_ruby.config.public_output_dir}" ],
                                     root: root.join(vite_ruby.config.public_dir),
                                     header_rules: [
                                       # rubocop:disable Style/StringHashKeys
                                       [ :all, { "Access-Control-Allow-Origin" => "*" } ]
                                       # rubocop:enable Style/StringHashKeys
                                     ]
      end
    end

    initializer "my_engine.vite_rails_engine.proxy" do |app|
      if vite_ruby.run_proxy?
        app.middleware.insert_before 0,
                                     ViteRuby::DevServerProxy,
                                     ssl_verify_none: true,
                                     vite_ruby: vite_ruby
      end
    end

    initializer "my_engine.vite_rails_engine.logger" do
      config.after_initialize do
        vite_ruby.logger = Rails.logger
      end
    end
  end
end
```
{:file='lib/my_engine/engine.rb'}

```json
{
  "all": {
    "sourceCodeDir": "app/frontend",
    "watchAdditionalPaths": [],
    "publicOutputDir": "my_engine-assets"
  },
  "development": {
    "autoBuild": true,
    "publicOutputDir": "my_engine-assets-dev",
    "port": 3036
  },
  "test": {
    "autoBuild": true,
    "publicOutputDir": "my_engine-assets-test",
    "port": 3037
  }
}
```
{:file='config/vite.json'}

```sh
rails g controller home index
```

```ruby
MyEngine::Engine.routes.draw do
  root "home#index"
end
```
{:file='config/routes.rb'}

```diff
<!DOCTYPE html>
<html>
  <head>
    <title>My engine</title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= yield :head %>
+   <%= vite_client_tag %>
+   <%= vite_javascript_tag 'application' %>
    <%= stylesheet_link_tag    "my_engine/application", media: "all" %>
  </head>
  <body>
    <%= yield %>
  </body>
</html>
```
{:file='app/views/layouts/my_engine/application.html.erb'}

```ruby
require "vite_rails/version"
require "vite_rails/tag_helpers"

module MyEngine
  module ApplicationHelper
    include ::ViteRails::TagHelpers

    def vite_manifest
      ::MyEngine::Engine.vite_ruby.manifest
    end
  end
end
```
{:file='app/helpers/my_engine/application_helper.rb'}

***Create host app***

```sh
rails new demo
```

```ruby
gem "my_engine", path: "path/to/engine"
```
{:file='Gemfile'}

```ruby
mount MyEngine::Engine => "/my_engine"
```
{:file='config/routes.rb'}


![](https://i.imgur.com/GdtXnER.png)


## References

- https://github.com/maglevhq/maglev-core