---
layout: post
title: Full text search with Thinking Sphinx
date: 2023-11-01 12:37 +0700
categories: Articles
tags: sphinx
---

ในหัวข้อนี้เราจะมาลองใช้ Sphinx เป็น full text search บน Rails กัน:

- Ruby 3.1.2
- Rails 7.0.4
- Sphinx 3.3.1
- Thinking Sphinx 5.4.0

## Getting Started
ทำการติดตั้ง [Sphinx](https://freelancing-gods.com/thinking-sphinx/v5/installing_sphinx.html) ทำตามขั้นตอนในนี้ได้เลย

สร้าง rails app โดยใช้ mysql เป็น database:

```bash
rails new shopshop --database=mysql
```

จากนั้นเพิ่มบรรทัดนี้ลงใน Gemfile:

```
gem 'thinking-sphinx'
```

แล้วเรียกคำสั่ง `bundle install`

ที่นี้เราจะสร้าง model ที่ต้องการจะใช้ค้นหา ในที่นี้ก็จะใช้เป็น product:

```bash
rails g scaffold Product name detail:text
rails db:migrate
rails s
```

เพิ่ม search ใน routes:

```ruby
Rails.application.routes.draw do
  resources :products do
    collection do
      get 'search'
    end
  end
end
```
{:file='config/routes.rb'}

เพิ่ม search ใน products controller:

```ruby
class ProductsController < ApplicationController
  ...
  def search
    @products = Product.search(params[:q])
    render :index
  end
  ...
end
```

เพิ่มฟอร์มสำหรับ search ในหน้า index ของ products:

```erb
<%= form_tag search_products_path, method: :get do %>
  <%= text_field_tag :q %>
  <%= submit_tag :search %>
<% end %>
```
{:file='app/views/products/index.html.erb'}

จากนั้นเราจะสร้างไฟล์เพื่อกำหนดว่าจะทำ index กับข้อมูลอะไรบ้างใน model นั้น โดยรูปแบบจะเป็นแบบนี้ `app/indices/[modelname]_index.rb`:

สร้างไฟล์ index ของ product ใน `app/indices/product_index.rb` และกำหนดตัวแปรที่จะทำ indexing:

```ruby
ThinkingSphinx::Index.define :product, with: :real_time do
  indexes name, sortable: true
  indexes detail
end
```
{:file='app/indices/product_index.rb'}

เรียกใช้คำสั่งเพื่อทำ index แล้วเริ่มใช้ Sphinx:

```bash
rails ts:configure
rails ts:stop
rails ts:index
rails ts:start
# or
rails ts:rebuild
```

ถ้าไม่อยากทำ indexing ด้วยมือทุกครั้งที่มีข้อมูลเพิ่ม ให้เราเพิ่มส่วนนี้ลงไปเพื่อจะได้ทำ indexing แบบ real time:

```ruby

class Product < ApplicationRecord
  ThinkingSphinx::Callbacks.append(self, behaviours: [:real_time])
end
```
{:file='app/models/product.rb'}

เพิ่มการตั้งค่า charset เพื่อให้รองรับการค้นหาด้วยอักษรภาษาไทย:

```yaml
development:
  charset_table: "0..9, A..Z->a..z, _, a..z, U+E00..U+E7F"
```
{:file='config/thinking_sphinx.yml'}

แต่การค้นหาภาษาไทยสำหรับ Sphinx จะค้นหาได้ไม่สมบูรณ์นัก เพราะไม่มีฟีเจอร์การตัดคำ ซึ่งภาษาไทยไม่ได้เว้นวรรคแบบภาษาอังกฤษ ซึ่งผมก็ได้สร้าง [gem](https://github.com/phuwanart/thbrk) สำหรับใช้ตัดคำเพื่อให้ค้นหาด้วย Sphinx ได้ เข้าไปดูแล้วทำตามได้เลย
