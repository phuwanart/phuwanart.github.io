---
layout: post
title: Rails Engine with Vite
categories:
- Posts
tags:
- rails
- rails engine
- vite
date: 2024-12-16 13:57 +0700
---

Rails engine คือ mountable app ที่อยู่ภายในแอปอีกที่ใหญ่กว่า — เหมาะกับการแพ็คฟีเจอร์ที่อยากใช้ซ้ำในหลายโปรเจกต์ (ระบบ admin, e-commerce module, CMS) เมื่อ engine ต้องมี frontend asset ของตัวเอง การเลือก [Vite](https://vite.dev/) แทน importmap หรือ jsbundling-rails ได้ HMR เร็ว ES modules ตรงไปตรงมา และเข้ากับ frontend framework อย่าง Vue/React ได้ดี

โพสต์นี้พาตั้ง engine ที่ build asset ของตัวเองด้วย Vite ไม่ชนกับ asset ของ host app แล้วต่อด้วย Tailwind CSS และ Vue พร้อมวิธีส่งข้อมูลจาก Rails ไป Vue component

## Prerequisites

- Ruby 3.2+
- Rails 7+
- Node 20+ และ Yarn
- ความคุ้นเคยกับ Rails engine และ Vite เบื้องต้น

## สร้าง engine

```sh
rails plugin new my_engine --mountable
```

flag `--mountable` ทำให้ engine มี namespace แยก (`MyEngine::`) และมี Rails app structure ของตัวเอง (controllers, views, routes, layouts) แทน plugin แบบธรรมดาที่แชร์ namespace กับ host app

## ติดตั้ง Vite

ประกาศ dependency ใน gemspec ของ engine:

```ruby
spec.add_dependency 'vite_rails'
spec.add_dependency 'vite_ruby'
```
{: file="my_engine.gemspec"}

ทั้งสอง gem ต้องเป็น runtime dependency เพราะ engine ใช้ `vite_rails` helper ใน view และ `vite_ruby` build asset ของตัวเอง ไม่ได้ใช้แค่ตอน dev

แก้ Rakefile ของ engine ให้โหลด vite task และตั้ง root ของ Vite ให้ชี้มาที่ engine:

```diff
 require "bundler/setup"

+require 'vite_ruby'
+ViteRuby.install_tasks
+ViteRuby.config.root # Ensure the engine is set as the root.

 APP_RAKEFILE = File.expand_path("test/dummy/Rakefile", __dir__)
 load "rails/tasks/engine.rake"

 load "rails/tasks/statistics.rake"

 require "bundler/gem_tasks"
```
{: file="Rakefile"}

บรรทัด `ViteRuby.config.root` สำคัญตรงที่บังคับ Vite ให้มอง engine root เป็นที่อยู่ของ source ไม่ใช่ dummy app ใน `test/dummy/` — ถ้าไม่มี Vite จะหา `vite.config.ts` ไม่เจอตอน task รัน

ติดตั้ง Vite ลง engine:

```sh
bundle exec vite install
```

## Engine middleware setup

ส่วนที่ซับซ้อนที่สุดของการเชื่อม Vite เข้ากับ engine — engine ต้อง serve static asset และ proxy ไปยัง Vite dev server เอง เพราะ host app ไม่รู้เรื่อง Vite ของ engine

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
{: file="lib/my_engine/engine.rb"}

แต่ละ initializer ทำอะไร:

- **`self.vite_ruby`** — สร้าง ViteRuby instance ที่ผูกกับ root ของ engine (ไม่ใช่ host app) ทำให้ asset path, manifest, dev server ทุกอย่างอ้างอิงโฟลเดอร์ของ engine
- **Initializer `static`** — เพิ่ม `Rack::Static` middleware เพื่อ serve asset ที่ Vite compile ไว้ใต้ `public/<output_dir>` ของ engine มี 2 branch:
  - **`public_file_server.enabled` เป็น true** — host app serve static เอง (dev mode ของ Rails หรือ prod ที่ไม่มี reverse proxy) ใส่ middleware ของเรา *หลัง* `ActionDispatch::Static` เพื่อให้ host app try ก่อน แล้วค่อย fallback มาที่ engine
  - **กรณีหลัง NGINX/Apache** — Rails ไม่ serve static เลย ใส่ middleware *ก่อน* index 0 เพื่อให้ Rack จับ asset request ของ engine ทันก่อน Rails routing
- **`Access-Control-Allow-Origin: *`** — เผื่อ host app และ engine asset อยู่คนละ origin (เช่นใช้ CDN หรือ subdomain) จะได้โหลด font/image cross-origin ได้
- **Initializer `proxy`** — ถ้า `bin/vite dev` กำลังรันอยู่ (`run_proxy?` คืน true) สอดแทรก `ViteRuby::DevServerProxy` เป็น middleware แรกเลย เพื่อ proxy request asset ไปที่ Vite dev server สำหรับ HMR
- **Initializer `logger`** — ทำให้ log ของ Vite ออกใน Rails log เดียวกัน อ่านง่ายตอน debug

## `vite.json`

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
{: file="config/vite.json"}

- **`sourceCodeDir: app/frontend`** — แยกออกจาก `app/javascript` ที่ Rails default ใช้กับ jsbundling/importmap เพื่อให้ชัดว่าโค้ดในนี้คุมโดย Vite
- **`publicOutputDir: my_engine-assets`** — สำคัญสุดของไฟล์นี้ — ตั้งชื่อ output folder ที่ unique ตามชื่อ engine ถ้าใช้ default (`vite`) จะชนกับ output ของ host app ที่ใช้ Vite อยู่ด้วย
- **`port` ต่างกันต่อ environment** — กัน port ชนกันถ้ามีหลาย engine หรือรัน test คู่กับ dev

## Controller, route, layout

สร้าง controller สำหรับลอง:

```sh
rails g controller home index
```

ตั้ง route ใน engine (อยู่ภายใต้ `MyEngine::Engine.routes`):

```ruby
MyEngine::Engine.routes.draw do
  root "home#index"
end
```
{: file="config/routes.rb"}

แก้ layout ให้โหลด vite client + entrypoint ของ engine:

```diff
 <!DOCTYPE html>
 <html>
   <head>
     <title>My engine</title>
     <%= csrf_meta_tags %>
     <%= csp_meta_tag %>
     <%= yield :head %>
+    <%= vite_client_tag %>
+    <%= vite_javascript_tag 'application' %>
     <%= stylesheet_link_tag    "my_engine/application", media: "all" %>
   </head>
   <body>
     <%= yield %>
   </body>
 </html>
```
{: file="app/views/layouts/my_engine/application.html.erb"}

`vite_client_tag` คือ script ของ Vite HMR (จะ render ออกมาเฉพาะตอน dev) ส่วน `vite_javascript_tag 'application'` ชี้ไปที่ `app/frontend/entrypoints/application.js`

## ApplicationHelper สำหรับ vite tag

engine ไม่ inherit helper จาก host app — ต้อง include `ViteRails::TagHelpers` เองและชี้ `vite_manifest` ไปที่ manifest ของ engine ไม่งั้น helper จะหา manifest ของ host app เจอ (และโหลด asset ผิด):

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
{: file="app/helpers/my_engine/application_helper.rb"}

## สร้าง host app และ mount engine

สร้างแอป demo สำหรับ mount engine:

```sh
rails new demo
```

เพิ่ม engine เป็น path gem:

```ruby
gem "my_engine", path: "path/to/engine"
```
{: file="Gemfile"}

mount engine ใน route ของ host app:

```ruby
mount MyEngine::Engine => "/my_engine"
```
{: file="config/routes.rb"}

รัน `bin/vite dev` ใน folder ของ engine แล้ว start host app — เปิด `http://localhost:3000/my_engine` จะเห็นหน้า home ของ engine พร้อม asset จาก Vite

![](https://i.imgur.com/GdtXnER.png)

## TailwindCSS

> ตัวอย่างต่อจากนี้ใช้ Tailwind v3 ตามช่วงเวลาที่เขียน post หากใช้ Tailwind v4 syntax `tailwind.config.js` และ `@tailwind` directive ต่างไป — ดูตัวอย่าง v4 ที่โพสต์ Preline หรือ Flowbite
{: .prompt-info }

ติดตั้ง Tailwind ใน engine:

```sh
yarn add -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

ตั้ง content path ให้ครอบไฟล์ของ engine ทั้งหมด:

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
{: file="tailwind.config.js"}

สร้าง CSS entrypoint:

```sh
touch app/frontend/entrypoints/application.css
```

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```
{: file="app/frontend/entrypoints/application.css"}

เพิ่ม `vite_stylesheet_tag` ใน layout:

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
+    <%= vite_stylesheet_tag 'application', data: { "turbo-track": "reload" } %>
     <%= stylesheet_link_tag    "my_engine/application", media: "all" %>
   </head>
   <body>
     <%= yield %>
   </body>
 </html>
```
{: file="app/views/layouts/my_engine/application.html.erb"}

ทดสอบด้วยการใส่ class ใน view:

```erb
<h1 class="font-bold text-4xl text-indigo-500">Home#index</h1>
<p>Find me in app/views/my_engine/home/index.html.erb</p>
```
{: file="app/views/my_engine/home/index.html.erb"}

![](https://i.imgur.com/OFoNtcK.png)

## Vue

ติดตั้ง Vue plugin:

```sh
yarn add -D vue @vitejs/plugin-vue
```

ใส่ plugin ใน Vite config:

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
{: file="vite.config.js"}

สร้าง single-file component:

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
{: file="app/frontend/components/App.vue"}

วาง mount point ใน view:

```erb
<div id="app"></div>
```
{: file="app/views/my_engine/home/index.html.erb"}

mount Vue app ใน entrypoint:

```js
import { createApp } from 'vue'
import App from '../components/App.vue'

const app = createApp(App)
app.mount('#app')
```
{: file="app/frontend/entrypoints/application.js"}

### ส่งข้อมูลจาก Rails ไป Vue

Vue mount หลัง DOM render แล้ว ไม่ได้รับ instance variable ของ Rails ตรงๆ pattern ที่นิยมคือ serialize ข้อมูลเป็น `data-*` attribute แล้ว Vue อ่านตอน mount

ใน component ประกาศ prop:

```vue
<template>
  <div>
    {% raw %}<h1>{{ msg }}</h1>{% endraw -%}
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
{: file="app/frontend/components/App.vue"}

ส่งข้อมูลจาก controller:

```ruby
module MyEngine
  class HomeController < ApplicationController
    def index
      @msg = "Hello Vue on Rails!"
    end
  end
end
```
{: file="app/controllers/my_engine/home_controller.rb"}

render เป็น `data-props`:

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
{: file="app/views/my_engine/home/index.html.erb"}

จะได้ markup ออกมาประมาณ:

```html
<div
    id="appProps"
    data-props="{&quot;msg&quot;:&quot;Hello Vue on Rails!&quot;}">
</div>
```

อ่าน prop ตอน mount Vue:

```js
import { createApp } from 'vue'
import App from '../components/App.vue'

const appProps = document.getElementById('appProps').dataset.props
const app = createApp(App, JSON.parse(appProps))
app.mount('#app')
```
{: file="app/frontend/entrypoints/application.js"}

ข้อจำกัดของ pattern นี้:

- ข้อมูลถูก serialize เป็น JSON string ครั้งเดียวตอน render ไม่มี reactivity กลับไปหา Rails
- ค่าที่ encode ไม่ได้เป็น JSON (Date instance, ActiveRecord object) ต้อง map เป็น primitive ก่อน
- ถ้าข้อมูลใหญ่ HTML จะบวมตามไปด้วย — ใช้ API call แทนถ้ามีข้อมูลเยอะ

![](https://i.imgur.com/FeJPJJ7.png)

## Troubleshooting

**`Vite Ruby: Manifest file not found`**
ยังไม่ได้รัน `bin/vite build` (สำหรับ production) หรือ `bin/vite dev` ยังไม่ขึ้น (สำหรับ dev) — เริ่ม dev server ใน folder ของ engine ก่อน start host app

**Asset 404 ทั้งที่ build ผ่าน**
ส่วนใหญ่ `publicOutputDir` ใน `vite.json` ชนกับ host app หรือ engine อื่น ตรวจให้ unique ต่อ engine

**`SSL_connect ... ssl handshake failed` ตอน Vite dev**
proxy ของ engine ใช้ `ssl_verify_none: true` อยู่แล้ว แต่ถ้ายังเจอ ตรวจว่า dev server รันด้วย HTTP (ไม่ใช่ HTTPS) — Vite default รัน HTTP

**HMR ไม่ทำงานในหน้าของ engine**
ต้องแน่ใจว่าใส่ `vite_client_tag` ใน layout ของ engine (ไม่ใช่ของ host app) และเปิด browser ไปที่ route ของ engine ผ่าน host app

## Conclusion

ตอนนี้ engine มี frontend stack เป็นของตัวเอง — Vite สำหรับ bundling, Tailwind สำหรับ styling, Vue สำหรับ component — และ mount เข้ากับ host app ได้โดยไม่กระทบ asset pipeline ของ host

วิธีนี้คุ้มเมื่อต้อง reuse engine ในหลายแอป (admin panel, white-label module, multi-tenant feature) แต่ถ้าทำแอปเดียว ใช้ Vite ตรงๆ ใน Rails app ง่ายกว่าและไม่ต้องเขียน middleware setup เอง

## References

- [vite_ruby documentation](https://vite-ruby.netlify.app/)
- [Getting Started with Engines (Rails Guides)](https://guides.rubyonrails.org/engines.html)
- <https://github.com/maglevhq/maglev-core>
- <https://primevise.com/blog/ruby-on-rails-boilerplate-vite-tailwind-stimulus>
- <https://dev.to/kevinluo201/use-viterails-to-use-vue-sfcsingle-file-component-vue-in-rails7-51bn>
