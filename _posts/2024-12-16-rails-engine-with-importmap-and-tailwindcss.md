---
layout: post
title: Rails Engine with Importmap and TailwindCSS
categories:
- Posts
tags:
- rails
- rails engine
- importmap
- tailwindcss
date: 2024-12-16 13:57 +0700
---

[Importmap](https://github.com/rails/importmap-rails) เป็น default frontend ของ Rails 7+ ที่ส่ง JavaScript เป็น ES module ตรงๆ ไปยัง browser ไม่ต้องมี Node, bundler, หรือ transpile step ทำให้ engine ที่ใช้ importmap ติดตั้งและ deploy ง่ายกว่า [แบบ Vite](/posts/rails-engine-with-vite/) — แลกกับการไม่มี HMR และไม่เหมาะกับ JS heavy framework แบบ Vue/React

โพสต์นี้พาทำ engine ชื่อ `blorgh` ที่มี importmap ของตัวเอง โหลด Turbo + Stimulus และ build Tailwind CSS แยกจาก host app

## Prerequisites

- Ruby 3.2+
- Rails 7+
- ความคุ้นเคยกับ Rails engine, importmap-rails และ Stimulus เบื้องต้น

## สร้าง engine

```sh
rails plugin new blorgh --mountable
```

`--mountable` ให้ engine มี namespace แยก (`Blorgh::`) และมี Rails app structure ของตัวเอง (controllers, views, routes, layouts)

## ติดตั้ง importmap + Hotwire

ประกาศใน gemspec ของ engine:

```ruby
spec.add_dependency 'importmap-rails'
spec.add_dependency 'turbo-rails'
spec.add_dependency 'stimulus-rails'
```
{: file="blorgh.gemspec"}

ทั้งสามต้องเป็น runtime dependency เพราะ engine ใช้ helper, asset path, และ importmap-rails ตอนรันจริง ไม่ได้ใช้แค่ตอนพัฒนา

## Engine initializer

```ruby
require "importmap-rails"
require "turbo-rails"
require "stimulus-rails"

module Blorgh
  class Engine < ::Rails::Engine
    isolate_namespace Blorgh

    initializer "blorgh.assets" do |app|
      app.config.assets.paths << root.join("app/javascript")
    end

    initializer "blorgh.importmap", before: "importmap" do |app|
      app.config.importmap.paths << root.join("config/importmap.rb")
      app.config.importmap.cache_sweepers << root.join("app/javascript")
    end
  end
end
```
{: file="lib/blorgh/engine.rb"}

แต่ละ initializer ทำอะไร:

- **`assets.paths << root.join("app/javascript")`** — เพิ่มโฟลเดอร์ JS ของ engine เข้า asset pipeline ของ host app ทำให้ importmap หา resolve `import "blorgh/application"` ไปที่ไฟล์จริงเจอ
- **`importmap.paths << root.join("config/importmap.rb")`** — register `importmap.rb` ของ engine เข้ากับ importmap ของ host app ตอน boot pin ของ engine จะรวมเข้ากับ pin ของ host app เป็นชุดเดียว
- **`cache_sweepers << root.join("app/javascript")`** — ให้ importmap reload cache เมื่อไฟล์ใน engine controllers folder เปลี่ยนใน dev mode (ไม่งั้นต้อง restart server ทุกครั้งที่แก้ controller)
- **`before: "importmap"`** — เรียก initializer นี้ก่อน initializer ของ importmap-rails เพื่อให้ path ของ engine ถูก register ทันก่อน importmap จะอ่าน config

## JavaScript entrypoint

ใช้ pattern เดียวกับ Rails default — แยก 3 ไฟล์:

```js
import "@hotwired/turbo-rails"
import "controllers"
```
{: file="app/javascript/blorgh/application.js"}

entrypoint หลัก โหลด Turbo และ trigger การ load controllers ทั้งหมด

```js
import { Application } from "@hotwired/stimulus"

const application = Application.start()

// Configure Stimulus development experience
application.debug = false
window.Stimulus = application

export { application }
```
{: file="app/javascript/blorgh/controllers/application.js"}

bootstrap Stimulus app instance ตั้ง `window.Stimulus` ไว้ debug จาก console ได้

```js
// Import and register all your controllers from the importmap via controllers/**/*_controller
import { application } from "controllers/application"
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading"
eagerLoadControllersFrom("controllers", application)
```
{: file="app/javascript/blorgh/controllers/index.js"}

eager load controller ทั้งหมดที่ pin ไว้ภายใต้ namespace `controllers`

## `config/importmap.rb` ของ engine

```ruby
pin "blorgh/application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js", preload: true
pin_all_from Blorgh::Engine.root.join("app/javascript/blorgh/controllers"), under: "controllers", to: "blorgh/controllers"
```
{: file="config/importmap.rb"}

- **`pin "blorgh/application"`** — pin entrypoint ของ engine ด้วย namespace `blorgh/` กันชนกับ `application` ของ host app
- **`pin "@hotwired/turbo-rails", to: "turbo.min.js"`** — ชี้ logical name ไปที่ asset ที่ turbo-rails gem provide ผ่าน asset pipeline
- **`pin_all_from ... under: "controllers", to: "blorgh/controllers"`** — pin ทุกไฟล์ใน `app/javascript/blorgh/controllers/` โดยใช้ logical namespace `controllers` แต่ resolve ไปที่ path จริง `blorgh/controllers/...` ตรงนี้แยกชัดระหว่าง name ที่โค้ดใช้กับ path จริงบน disk

## Custom helper สำหรับ importmap ของ engine

มาตรฐาน `javascript_importmap_tags` ใช้ `Rails.application.importmap` ของ host app ซึ่งจะรวม pin ของ engine เข้ามาแล้วก็จริง แต่ถ้าอยากให้ engine render importmap ของตัวเองเป็นบล็อกแยก (แยก namespace ชัดเจน เผื่อหลาย engine แต่ละตัวมี Hotwire version ต่างกัน) ต้องสร้าง `Importmap::Map` instance ของ engine เอง:

```ruby
module Blorgh
  class Configuration
    attr_accessor :importmap

    def initialize
      @importmap = Importmap::Map.new
      @importmap.draw(Engine.root.join("config/importmap.rb"))
    end
  end
end
```
{: file="lib/blorgh/configuration.rb"}

ผูก configuration เข้ากับ namespace ของ engine แบบ Configurable pattern:

```ruby
require "blorgh/version"
require "blorgh/engine"
require "blorgh/configuration"

module Blorgh
  class << self
    attr_writer :configuration

    def configuration
      @configuration ||= Configuration.new
    end

    def configure
      yield(configuration) if block_given?
    end
  end
end
```
{: file="lib/blorgh.rb"}

แล้วเขียน helper ที่ render importmap ของ engine ออกมาเองในสามส่วน:

```ruby
def blorgh_importmap_tags(entry_point = "application")
  safe_join [
    javascript_inline_importmap_tag(Blorgh.configuration.importmap.to_json(resolver: self)),
    javascript_importmap_module_preload_tags(Blorgh.configuration.importmap),
    javascript_import_module_tag(entry_point)
  ].compact, "\n"
end
```
{: file="app/helpers/blorgh/application_helper.rb"}

- **`javascript_inline_importmap_tag`** — render `<script type="importmap">{...}</script>` ที่บอก browser ว่า bare specifier (`"controllers"`, `"@hotwired/turbo-rails"`) map ไปที่ URL ไหน
- **`javascript_importmap_module_preload_tags`** — render `<link rel="modulepreload">` ให้ทุก pin ที่ตั้ง `preload: true` browser จะ fetch ขนานกันแทนรอ chain
- **`javascript_import_module_tag(entry_point)`** — render `<script type="module">import "blorgh/application"</script>` เพื่อ trigger การโหลด entrypoint

## Layout

```diff
 <!DOCTYPE html>
 <html>
   <head>
     <title>Blorgh</title>
     <%= csrf_meta_tags %>
     <%= csp_meta_tag %>
     <%= yield :head %>
     <%= stylesheet_link_tag    "blorgh/application", media: "all" %>
+    <%= blorgh_importmap_tags %>
   </head>
   <body>
     <%= yield %>
   </body>
 </html>
```
{: file="app/views/layouts/blorgh/application.html.erb"}

## Controller, route, ทดสอบ Stimulus

รัน command ต่อไปนี้จากใน folder ของ engine:

```sh
rails g controller home index
```

ตั้ง root route:

```ruby
Blorgh::Engine.routes.draw do
  root "home#index"
end
```
{: file="config/routes.rb"}

สร้าง Stimulus controller ทดสอบ:

```js
import { Controller } from "@hotwired/stimulus"

// Connects to data-controller="engine"
export default class extends Controller {
  connect() {
    console.log("Connected")
    this.element.textContent = "Hello World! This is a Javascript from the Engine"
  }
}

console.log("Loaded")
```
{: file="app/javascript/blorgh/controllers/engine_controller.js"}

ผูกกับ markup:

```erb
<h1>Engine's Stimulus controller (Engine)</h1>
<div data-controller="engine"></div>
```
{: file="app/views/blorgh/home/index.html.erb"}

## สร้าง host app และ mount

```sh
rails new demo
```

```ruby
gem "blorgh", path: "path/to/engine"
```
{: file="Gemfile"}

```ruby
mount Blorgh::Engine => "/blorgh"
```
{: file="config/routes.rb"}

เปิด browser ที่ `http://localhost:3000/blorgh` จะเห็น Stimulus controller ทำงานและเปลี่ยน text เป็น "Hello World! This is a Javascript from the Engine"

![](https://i.imgur.com/lTQZedb.png)

## Tailwind CSS

วิธีนี้ engine build CSS ของตัวเองแยกออกมาเป็น `app/assets/builds/blorgh.css` แล้ว host app จะ pick up ผ่าน asset pipeline ด้วย `stylesheet_link_tag "blorgh"` ตามปกติ — ไม่ต้องตั้งค่าฝั่ง host app เพิ่ม

เพิ่ม dependency ใน gemspec:

```ruby
spec.add_dependency "tailwindcss-rails"
```
{: file="blorgh.gemspec"}

แก้ layout ให้โหลด stylesheet ของ engine:

```erb
<%= stylesheet_link_tag "blorgh", "data-turbo-track": "reload" %>
```
{: file="app/views/layouts/blorgh/application.html.erb"}

ทดสอบใส่ class:

```erb
<h1 class="text-red-500">Engine's Stimulus controller (Engine)</h1>
<div data-controller="engine"></div>
```
{: file="app/views/blorgh/home/index.html.erb"}

Copy template ของ Tailwind config และ entrypoint CSS มาจาก gem ของ tailwindcss-rails:

```sh
cp $(bundle show tailwindcss-rails)/lib/install/tailwind.config.js config/tailwind.config.js
cp $(bundle show tailwindcss-rails)/lib/install/application.tailwind.css app/assets/stylesheets/blorgh/application.tailwind.css
```

ที่ใช้ `cp` แทน `rails tailwindcss:install` เพราะ generator ของ gem มุ่งติดตั้งใน Rails app เต็มรูปแบบ ไม่ใช่ใน engine — copy template มา edit เองตรงไปตรงมากว่า

Build CSS:

```sh
$(bundle show tailwindcss-ruby)/exe/tailwindcss -i app/assets/stylesheets/blorgh/application.tailwind.css -o app/assets/builds/blorgh.css -c config/tailwind.config.js --minify
```

Watch ตอนพัฒนา:

```sh
$(bundle show tailwindcss-ruby)/exe/tailwindcss -i app/assets/stylesheets/blorgh/application.tailwind.css -o app/assets/builds/blorgh.css -c config/tailwind.config.js --minify -w
```

ตรวจ `content` array ใน `config/tailwind.config.js` ให้ครอบไฟล์ของ engine ครบ (`app/views/**/*.html.erb`, `app/javascript/**/*.js`, ฯลฯ) ไม่งั้น class จะถูก purge ออกตอน build

![](https://i.imgur.com/FRCpQij.png)

## Troubleshooting

**Stimulus controller ไม่เชื่อม (`data-controller="engine"` ไม่ทำงาน)**
ตรวจ console ใน browser ว่ามี error `Cannot find module 'controllers/engine_controller'` หรือไม่ ถ้ามีแสดงว่า `pin_all_from` ใน `config/importmap.rb` ไม่ตรงกับ path จริง หรือลืม `eagerLoadControllersFrom("controllers", ...)` ใน `controllers/index.js`

**`Uncaught Error: Cannot find module` ตอนโหลดหน้า**
มี `import` ในโค้ดที่ยังไม่ได้ pin เพิ่มใน `config/importmap.rb` ของ engine และ restart server

**Tailwind class ใน view ของ engine ไม่ generate**
`content` ใน `config/tailwind.config.js` ครอบไม่ทั่ว — ตรวจให้รวม `./app/views/**/*.html.erb`, `./app/helpers/**/*.rb`, `./app/javascript/**/*.js` ของ engine

**Importmap ของ engine ชนกับ host app**
ถ้าใช้ `blorgh_importmap_tags` แล้วยังเจอ ตรวจว่า host app's layout ไม่ได้ render `javascript_importmap_tags` ของตัวเองในหน้าของ engine (อาจซ้อนกัน) — engine มี layout แยก ใช้ `app/views/layouts/blorgh/application.html.erb` ของตัวเองโดย default

## Conclusion

วิธีนี้ทำให้ engine มี JS และ CSS เป็นของตัวเองโดยไม่ต้องพึ่ง Node toolchain — เหมาะกับ engine ที่ Hotwire-centric (Turbo + Stimulus + เปลี่ยน DOM บางส่วน) และอยากเก็บ build pipeline ให้น้อยที่สุด

จะเลือก importmap หรือ [Vite](/posts/rails-engine-with-vite/) ขึ้นอยู่กับ:

- **เลือก importmap ถ้า** — JS น้อย, ใช้ Hotwire เป็นหลัก, ไม่อยากมี Node ใน toolchain, deploy ง่ายเป็น priority
- **เลือก Vite ถ้า** — ใช้ Vue/React/Svelte, ต้องการ HMR, มี dependency JS เยอะที่ต้อง bundle, transpile TypeScript/JSX

## References

- [importmap-rails README](https://github.com/rails/importmap-rails)
- [tailwindcss-rails README](https://github.com/rails/tailwindcss-rails)
- [Getting Started with Engines (Rails Guides)](https://guides.rubyonrails.org/engines.html)
- <https://mariochavez.io/desarrollo/2023/08/23/working-with-rails-engines-importmap-tailwindcss/>
- <https://stackoverflow.com/questions/71232601/how-to-use-tailwind-css-gem-in-a-rails-7-engine>
- <https://radanskoric.com/articles/rails-assets-combine-importmaps>
