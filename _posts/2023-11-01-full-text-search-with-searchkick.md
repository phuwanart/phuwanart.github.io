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

มาถึงขั้นนี้แล้วก็เหมือนว่าทุกอย่างจะใช้ได้ดี ที่นี้ลองใส่ข้อมูลที่เป็นภาษาไทยลงไป:

แล้วก็ลองค้นหาดู ก็จะเห็นว่าได้ผลออกมา:

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
                 }
               }
             }
end
```
{:file='app/models/product.rb'}

```bash
rails searchkick:reindex CLASS=Product
```
