---
layout: post
title: Rails Engine with Importmap and TailwindCSS
categories:
- Notes
tags:
- rails
- rails engine
- importmap
- tailwindcss
date: 2024-12-16 13:57 +0700
---
## Create engine

```bash
rails plugin new blorgh --mountable
```

## Importmap configuration

```ruby
spec.add_dependency 'importmap-rails'
spec.add_dependency 'turbo-rails'
spec.add_dependency 'stimulus-rails'
```
{:file='blorgh.gemspec'}

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
{:file='lib/blorgh/engine.rb'}


```js
import "@hotwired/turbo-rails"
import "controllers"
```
{:file='app/javascript/blorgh/application.js'}

```js
import { Application } from "@hotwired/stimulus"

const application = Application.start()

// Configure Stimulus development experience
application.debug = false
window.Stimulus = application

export { application }
```
{:file='app/javascript/blorgh/controllers/application.js'}

```js
// Import and register all your controllers from the importmap via controllers/**/*_controller
import { application } from "controllers/application"
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading"
eagerLoadControllersFrom("controllers", application)
```
{:file='app/javascript/blorgh/controllers/index.js'}

```ruby
pin "blorgh/application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js", preload: true
pin_all_from Blorgh::Engine.root.join("app/javascript/blorgh/controllers"), under: "controllers", to: "blorgh/controllers"
```
{:file='config/importmap.rb'}

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
{:file='lib/blorgh/configuration.rb'}

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
{:file='lib/blorgh.rb'}

```ruby
def blorgh_importmap_tags(entry_point = "application")
  safe_join [
    javascript_inline_importmap_tag(Blorgh.configuration.importmap.to_json(resolver: self)),
    javascript_importmap_module_preload_tags(Blorgh.configuration.importmap),
    javascript_import_module_tag(entry_point)
  ].compact, "\n"
end
```
{:file='app/helpers/blorgh/application_helper.rb'}

```diff
<!DOCTYPE html>
<html>
  <head>
    <title>Blorgh</title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= yield :head %>
    <%= stylesheet_link_tag    "blorgh/application", media: "all" %>
+   <%= blorgh_importmap_tags %>
  </head>
  <body>
    <%= yield %>
  </body>
</html>
```
{:file='app/views/layouts/blorgh/application.html.erb'}

```sh
rails g controller home index
```
{:file='/blorgh'}

```ruby
Blorgh::Engine.routes.draw do
  root "home#index"
end
```
{:file='config/routes.rb'}

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
{:file='app/javascript/blorgh/controllers/engine_controller.js'}

```erb
<h1>Engine's Stimulus controller (Engine)</h1>
<div data-controller="engine"></div>
```
{:file='app/views/blorgh/home/index.html.erb'}

***Create host app***

```sh
rails new demo
```

```ruby
gem "blorgh", path: "path/to/engine"
```
{:file='Gemfile'}

```ruby
mount Blorgh::Engine => "/blorgh"
```
{:file='config/routes.rb'}

open browser: `http://localhost:3000/blorgh`


![](https://i.imgur.com/lTQZedb.png)


## TailwindCSS configuration

```ruby
spec.add_dependency "tailwindcss-rails"
```
{:file='blorgh.gemspec'}

```erb
<%= stylesheet_link_tag "blorgh", "data-turbo-track": "reload" %>
```
{:file='app/views/layouts/blorgh/application.html.erb'}

```erb
<h1 class="text-red-500">Engine's Stimulus controller (Engine)</h1>
<div data-controller="engine"></div>
```
{:file='app/views/blorgh/home/index.html.erb'}

```sh
cp $(bundle show tailwindcss-rails)/lib/install/tailwind.config.js config/tailwind.config.js
```

```sh
cp $(bundle show tailwindcss-rails)/lib/install/application.tailwind.css app/assets/stylesheets/blorgh/application.tailwind.css
```

Build engine Tailwind CSS

```sh
$(bundle show tailwindcss-ruby)/exe/tailwindcss -i app/assets/stylesheets/blorgh/application.tailwind.css -o app/assets/builds/blorgh.css -c config/tailwind.config.js --minify
```

Watch and build engine Tailwind CSS on file changes

```sh
$(bundle show tailwindcss-ruby)/exe/tailwindcss -i app/assets/stylesheets/blorgh/application.tailwind.css -o app/assets/builds/blorgh.css -c config/tailwind.config.js --minify -w
```


![](https://i.imgur.com/FRCpQij.png)


## References

- https://mariochavez.io/desarrollo/2023/08/23/working-with-rails-engines-importmap-tailwindcss/
- https://stackoverflow.com/questions/71232601/how-to-use-tailwind-css-gem-in-a-rails-7-engine
- https://radanskoric.com/articles/rails-assets-combine-importmaps
