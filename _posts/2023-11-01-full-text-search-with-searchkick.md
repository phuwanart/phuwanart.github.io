---
layout: post
title: Full text search with Searchkick
date: 2023-11-01 13:16 +0700
categories: Articles
tags: [opensearch, searchkick]
---

## Getting Started

ติดตั้ง [Elasticsearch](https://www.elastic.co/downloads/elasticsearch) หรือ [OpenSearch](https://opensearch.org/downloads.html) สำหรับคนที่ใช้ macOS สามารถติดตั้งโดยใช้ [brew](https://brew.sh/) ได้เลย:

```bash
brew install elastic/tap/elasticsearch-full
brew services start elasticsearch-full
# or
brew install opensearch
brew services start opensearch
```

จากนั้นเพิ่มบรรทัดเหล่านี้เข้าไปใน Gemfile:

```ruby
gem "searchkick"

gem "elasticsearch"   # select one
gem "opensearch-ruby" # select one
```
{:file='Gemfile'}

และเพิ่ม searchkick เข้าไปใน model ที่ต้องการจะค้นหา:

```ruby
class Product < ActiveRecord::Base
  searchkick
end
```
{:file='app/models/product.rb'}

เข้าไปอ่าน [Searchkick](https://github.com/ankane/searchkick) เพิ่มเติม ดูวิธีการใช้งานและ feature ที่มีให้ใช้

## Demo

ต่อไปเราจะไปลองทำ demo ง่าย ๆ กัน:

```bash
rails new store
```

จากนั้นก็สร้างโมเดล product ซึ่งขอลัด ๆ ใช้ scaffold ไปก่อน:

```bash
rails g scaffold Product name detail:text
rails db:migrate
rails s
```

สร้างตัวอย่างข้อมูลใน:

```ruby
Product.create(name: '', detail: '')
```
{:file='db/seeds.rb'}

จากนั้นก็รันคำสั่งเพื่อเพิ่มข้อมูลเข้าไป:

```bash
rails db:seed
```
เปิด http://localhost:3000/products:


![16988928511916](https://i.imgur.com/Z3dnbiv.png)


หลังจากนี้จะ implement search ให้กับ Product ก่อนอื่นก็เพิ่ม routes search:

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

เพิ่ม form การ search:

```erb
<%= form_tag search_products_path, method: :get do %>
  <%= text_field_tag :q %>
  <%= submit_tag :search %>
<% end %>
```
{:file='app/views/products/index.html.erb'}

จะได้ form มาแบบนี้:


![16988929644838](https://i.imgur.com/paxMUFn.png)


แล้วก็ไป implement search method:

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
{:file='app/controllers/products_controller.rb'}

ตอนนี้ทำง่าย ๆ ไปก่อน แล้วไปลองดูผลงานกัน:


![16988930135463](https://i.imgur.com/tLidcrB.png)


มาถึงขั้นนี้แล้วก็เหมือนว่าทุกอย่างจะใช้ได้ดี ที่นี้ลองใส่ข้อมูลที่เป็นภาษาไทยลงไป:


![16988931025125](https://i.imgur.com/18Grhkl.png)


แล้วก็ลองค้นหาดู ก็จะเห็นว่าได้ผลออกมา:


![16988931625161](https://i.imgur.com/i7f48aY.png)


เกือบดีละ ที่นี้ลองค้นหาแค่บางส่วนของคำ:

![16988932201872](https://i.imgur.com/YS24nY5.png)


หาไม่เจอ 😓 มันเป็นปัญหาอย่างเดียวกับที่เคยเจอตอนใช้ [Thinking Sphinx](https://freelancing-gods.com/thinking-sphinx) เป็นเรื่องการตัดคำภาษาไทยอีกแล้ว

**อัพเดท 2020:** จากนั้นแก้ไข searchkick ใน model โดยการเพิ่ม mapping สำหรับภาษาไทยเข้าไป ซึ่งจะต้องกำหนดว่าจะให้ค้นหาใน column ไหนได้บ้าง อย่างในตัวอย่างคือให้ค้นหาที่ name และ detail ต่างจากเมื่อก่อนที่กำหนดแต่ teken เป็น thai ก็พอ:

```ruby
class Product < ApplicationRecord
  searchkick merge_mappings: true,
             settings: {
               analysis: {
                 analyzer: {
                   thai_analyzer: {
                     tokenizer: 'thai'
                   }
                 }
               }
             },
             mappings: {
               properties: {
                 name: {
                   type: 'keyword',
                   fields: {
                     analyzed: {
                       type: 'text',
                       analyzer: 'thai_analyzer'
                     }
                   }
                 },
                 detail: {
                   type: 'keyword',
                   fields: {
                     analyzed: {
                       type: 'text',
                       analyzer: 'thai_analyzer'
                     }
                   }
                 }
               }
             }
end
```
{:file='app/models/product.rb'}

จากนั้น reindex อีกครั้ง:

```bash
rails searchkick:reindex CLASS=Product
```

ลองค้นหาผลลัพท์ดู:

![16988932763830](https://i.imgur.com/q7vLWyb.png)

เป็นอันเรียบร้อย ทำให้ค้นการแบบตัดคำภาษาไทยได้แล้ว 😉

## Conclusion

Elasticsearch เป็น search engine ที่ดี และ Searchkick ก็ทำให้มันใช้ง่าย ๆ กับ rails model และตอนนี้ก็หาวิธีการให้ค้นหาภาษาไทยได้อย่างมีประสิทธิภาพได้แล้ว เพราะงั้นงานต่อไปได้ใช้อย่างแน่นอน

**อัพเดท 2020:** Elasticsearch ไม่สามารถค้นหาภาษาไทยที่เป็นทับศัพท์ได้ น่าจะเพราะการตัดคำนั่นแหละ หากจะเลือกใช้ก็อย่าลืมดูเรื่องนี้ด้วย