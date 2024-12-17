---
layout: post
title: Rails Engine with Vite
categories: Posts
tags:
- rails
- rails engine
- vite
date: 2024-12-16 13:57 +0700
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


## TailwindCSS configuration

```sh
yarn add -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./app/views/**/*.rb",
    "./app/views/**/*.html.erb",
    "./app/views/layouts/*.html.erb",
    "./app/helpers/**/*.rb",
    "./app/assets/stylesheets/**/*.css",
    "./app/frontend/**/*.js",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```
{:file='tailwind.config.js'}

```sh
touch app/frontend/entrypoints/application.css
```

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```
{:file='app/frontend/entrypoints/application.css'}

```diff
<!DOCTYPE html>
<html>
  <head>
    <title>My engine</title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= yield :head %>
    <%= vite_client_tag %>
    <%= vite_javascript_tag 'application' %>
+   <%= vite_stylesheet_tag 'application', data: { "turbo-track": "reload" } %>
    <%= stylesheet_link_tag    "my_engine/application", media: "all" %>
  </head>
  <body>
    <%= yield %>
  </body>
</html>
```
{:file='app/views/layouts/my_engine/application.html.erb'}

```erb
<h1 class="font-bold text-4xl text-indigo-500">Home#index</h1>
<p>Find me in app/views/my_engine/home/index.html.erb</p>
```
{:file='app/views/my_engine/home/index.html.erb'}


![](https://i.imgur.com/OFoNtcK.png)

## Vue configuration

```sh
yarn add -D vue @vitejs/plugin-vue
```

```js
import { defineConfig } from 'vite'
import RubyPlugin from 'vite-plugin-ruby'
import vue from '@vitejs/plugin-vue' // <-------- add this

export default defineConfig({
  plugins: [
    RubyPlugin(),
    vue() // <-------- add this
  ],
})
```

```vue
<template>
  <div>
    <h1>Vue App!</h1>
  </div>
</template>

<script setup>

</script>

<style lang="css" scoped>
  h1 {
    color: red;
  }
</style>
```
{:file='app/frontend/components/App.vue'}


```erb
<div id="app"></div>
```
{:file='app/views/my_engine/home/index.html.erb'}

```js
import { createApp } from 'vue'
import App from '../components/App.vue'

const app = createApp(App)
app.mount('#app')
```
{:file='app/frontend/entrypoints/application.js'}

### Pass data from Rails to Vue components

```vue
<template>
  <div>
    <h1>{{ msg }}</h1>
  </div>
</template>

<script setup>
defineProps({
  msg: String
})
</script>

<style lang="css" scoped>
  h1 {
    color: red;
  }
</style>
```
{:file='app/frontend/components/App.vue'}

```ruby
module MyEngine
  class HomeController < ApplicationController
    def index
      @msg = "Hello Vue on Rails!"
    end
  end
end
```
{:file='app/controllers/my_engine/home_controller.rb'}

```erb
<%=
  content_tag(
    :div,
    id: 'appProps',
    data: {
      props: {
        msg: @msg
      }
    }.as_json
  ) {}
%>
<div id="app"></div>
```
{:file='app/views/my_engine/home/index.html.erb'}

it will render the div below on the page.

```html
<div 
    id="appProps" 
    data-props="{&quot;msg&quot;:&quot;Hello Vue on Rails!&quot;}">
</div>
```

```js
import { createApp } from 'vue'
import App from '../components/App.vue'

const appProps = document.getElementById('appProps').dataset.props
const app = createApp(App, JSON.parse(appProps))
app.mount('#app')
```
{:file='app/frontend/entrypoints/application.js'}


![](https://i.imgur.com/FeJPJJ7.png)


## References

- https://github.com/maglevhq/maglev-core
- https://primevise.com/blog/ruby-on-rails-boilerplate-vite-tailwind-stimulus
- https://dev.to/kevinluo201/use-viterails-to-use-vue-sfcsingle-file-component-vue-in-rails7-51bn