---
layout: post
title: Full text search with Thinking Sphinx
date: 2022-10-04 00:00 +0700
categories: Posts
tags: sphinx rails search thai
---

[Sphinx](https://sphinxsearch.com/) เป็น full-text search engine ที่รันแยกจาก database — ทำ index จากข้อมูลใน table แล้วตอบ search query ได้เร็วกว่า `LIKE '%foo%'` ของ SQL มาก [Thinking Sphinx](https://freelancing-gods.com/thinking-sphinx/) เป็น Rails wrapper ที่ทำให้ใช้ Sphinx ผ่าน ActiveRecord ได้สะดวก

ในหัวข้อนี้เราจะมาลองใช้ Sphinx เป็น full text search บน Rails กัน

## Prerequisites

- Ruby 3.1.2
- Rails 7.0.4
- Sphinx 3.3.1
- Thinking Sphinx 5.4.0
- MySQL (ใช้เป็น datastore)

## Getting Started

ทำการติดตั้ง [Sphinx](https://freelancing-gods.com/thinking-sphinx/v5/installing_sphinx.html) ทำตามขั้นตอนในนี้ได้เลย

สร้าง Rails app โดยใช้ MySQL เป็น database:

```sh
rails new shopshop --database=mysql
```
{: file="Local Machine"}

จากนั้นเพิ่มบรรทัดนี้ลงใน `Gemfile`:

```ruby
gem "thinking-sphinx"
```
{: file="Gemfile"}

แล้วเรียกคำสั่ง `bundle install`

ที่นี้เราจะสร้าง model ที่ต้องการจะใช้ค้นหา ในที่นี้ก็จะใช้เป็น product:

```sh
rails g scaffold Product name detail:text
rails db:migrate
rails s
```

เพิ่ม search ใน routes:

```ruby
Rails.application.routes.draw do
  resources :products do
    collection do
      get "search"
    end
  end
end
```
{: file="config/routes.rb"}

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
{: file="app/controllers/products_controller.rb"}

`Product.search(query)` คือ method ที่ Thinking Sphinx เพิ่มให้ — ส่ง query ไปยัง Sphinx แล้วคืน ActiveRecord collection ที่ match ตาม relevance ranking

เพิ่มฟอร์มสำหรับ search ในหน้า index ของ products:

```erb
<%= form_tag search_products_path, method: :get do %>
  <%= text_field_tag :q %>
  <%= submit_tag :search %>
<% end %>
```
{: file="app/views/products/index.html.erb"}

## กำหนด index

จากนั้นเราจะสร้างไฟล์เพื่อกำหนดว่าจะทำ index กับข้อมูลอะไรบ้างใน model นั้น โดยรูปแบบจะเป็นแบบนี้ `app/indices/[modelname]_index.rb`:

สร้างไฟล์ index ของ product ใน `app/indices/product_index.rb` และกำหนดตัวแปรที่จะทำ indexing:

```ruby
ThinkingSphinx::Index.define :product, with: :real_time do
  indexes name, sortable: true
  indexes detail
end
```
{: file="app/indices/product_index.rb"}

`with: :real_time` บอก Sphinx ให้ใช้ real-time index — Sphinx เก็บ index ใน memory และอัปเดตทันทีเมื่อ record เปลี่ยน (insert/update/delete) ทางเลือกคือ SQL-backed index (`with: :active_record` หรือ default) ที่ต้อง rebuild ทุกครั้งเมื่อข้อมูลเปลี่ยน — เร็วกว่าตอน search แต่ข้อมูลใหม่ไม่ปรากฏจนกว่าจะ rebuild

`sortable: true` ที่ `name` ทำให้ใช้ `order(name: :asc)` กับ result ได้ (Sphinx ต้องเก็บ string เป็น attribute เพิ่มเพื่อ sort)

## เริ่มใช้ Sphinx

เรียกใช้คำสั่งเพื่อทำ index แล้วเริ่มใช้ Sphinx:

```sh
rails ts:configure
rails ts:stop
rails ts:index
rails ts:start
# or
rails ts:rebuild
```

แต่ละ task ทำอะไร:

- **`ts:configure`** — generate `config/<env>.sphinx.conf` จาก index file ใน `app/indices/`
- **`ts:stop`** — หยุด Sphinx daemon (ถ้ารันอยู่)
- **`ts:index`** — สั่ง Sphinx build index file จาก database
- **`ts:start`** — start Sphinx daemon ขึ้นมารอรับ query
- **`ts:rebuild`** — รวมทั้ง 4 ขั้นข้างบนในคำสั่งเดียว สะดวกตอน dev

## Auto-index เมื่อ record เปลี่ยน

ถ้าไม่อยากทำ indexing ด้วยมือทุกครั้งที่มีข้อมูลเพิ่ม ให้เราเพิ่มส่วนนี้ลงไปเพื่อจะได้ทำ indexing แบบ real time:

```ruby
class Product < ApplicationRecord
  ThinkingSphinx::Callbacks.append(self, behaviours: [:real_time])
end
```
{: file="app/models/product.rb"}

`Callbacks.append` ผูก ActiveRecord callback (`after_save`, `after_destroy`) ให้อัปเดต real-time index ของ Sphinx ทันทีที่ record เปลี่ยน ไม่ต้อง rebuild manual

## ค้นหาภาษาไทย

ปกติ Sphinx tokenize string โดยตัดที่ space ซึ่งใช้ไม่ได้กับภาษาไทย — ขั้นต่ำที่สุดต้องเพิ่ม Unicode range ของอักษรไทย (`U+0E00..U+0E7F`) ใน `charset_table` ให้ Sphinx รู้จัก:

```yaml
development:
  charset_table: "0..9, A..Z->a..z, _, a..z, U+E00..U+E7F"
```
{: file="config/thinking_sphinx.yml"}

> การค้นหาภาษาไทยสำหรับ Sphinx จะค้นหาได้ไม่สมบูรณ์นัก เพราะไม่มีฟีเจอร์การตัดคำ ซึ่งภาษาไทยไม่ได้เว้นวรรคแบบภาษาอังกฤษ ซึ่งผมก็ได้สร้าง [gem `thbrk`](https://github.com/phuwanart/thbrk) สำหรับใช้ตัดคำเพื่อให้ค้นหาด้วย Sphinx ได้ เข้าไปดูแล้วทำตามได้เลย
{: .prompt-tip }

## ทางเลือกอื่นๆ

โพสต์นี้เขียนสมัย 2022 ปัจจุบัน Sphinx ยังใช้ได้ดีในงาน legacy แต่มีทางเลือกที่ active กว่า:

- **[Searchkick](https://github.com/ankane/searchkick) + Elasticsearch/OpenSearch** — modern stack ที่ scale ได้ดี มี ecosystem ใหญ่ ดูตัวอย่างที่ [OpenSearch setup with Docker Compose and Kamal](/posts/opensearch-setup-with-docker-compose-and-kamal-deploy/)
- **[pg_search](https://github.com/Casecommons/pg_search)** — full-text search ใน PostgreSQL ไม่ต้องลง service แยก เหมาะกับ app เล็กถึงกลางที่ใช้ Postgres อยู่แล้ว
- **[Meilisearch](https://www.meilisearch.com/) / [Typesense](https://typesense.org/)** — search engine รุ่นใหม่ ติดตั้งง่าย มี typo-tolerance built-in

Sphinx ยังเหมาะถ้า — มี legacy code base ที่ใช้ Sphinx อยู่แล้ว, ต้องการ performance สูงและคุม config ระดับลึก, หรือไม่อยากเพิ่ม dependency เป็น JVM (Elasticsearch/OpenSearch)

## References

- [Thinking Sphinx documentation](https://freelancing-gods.com/thinking-sphinx/)
- [Sphinx documentation](https://sphinxsearch.com/docs/sphinx3.html)
- [thbrk gem (Thai word breaker)](https://github.com/phuwanart/thbrk)
