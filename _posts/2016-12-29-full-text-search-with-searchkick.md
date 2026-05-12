---
layout: post
title: Full text search with Searchkick
date: 2016-12-29 00:00 +0700
categories: Posts
tags:
- opensearch
- elasticsearch
- searchkick
- rails
- search
- thai
---

[Searchkick](https://github.com/ankane/searchkick) เป็น Rails wrapper สำหรับ Elasticsearch/OpenSearch ที่ทำให้เพิ่ม full-text search ลง model ได้ด้วย macro `searchkick` บรรทัดเดียว และจัดการ index lifecycle (auto-reindex, async, mapping) ให้อัตโนมัติ

> Elasticsearch เปลี่ยน license เป็น SSPL ตั้งแต่ปี 2021 ไม่ใช่ open-source ตามนิยาม OSI แล้ว AWS จึง fork ออกมาเป็น **OpenSearch** ภายใต้ Apache 2.0 — Searchkick รองรับทั้งสองตัวด้วย client gem คนละตัว (`elasticsearch` หรือ `opensearch-ruby`)
{: .prompt-info }

## Prerequisites

- Ruby + Rails
- Elasticsearch หรือ OpenSearch รันอยู่ — บนเครื่อง dev จะใช้ Homebrew (วิธีในโพสต์นี้) หรือ Docker Compose ก็ได้ ดู [OpenSearch setup with Docker Compose and Kamal](/posts/opensearch-setup-with-docker-compose-and-kamal-deploy/) สำหรับ Docker Compose แบบ 2-node cluster

## Getting Started

ติดตั้ง [Elasticsearch](https://www.elastic.co/downloads/elasticsearch) หรือ [OpenSearch](https://opensearch.org/downloads.html) สำหรับคนที่ใช้ macOS สามารถติดตั้งโดยใช้ [brew](https://brew.sh/) ได้เลย:

```sh
brew install elastic/tap/elasticsearch-full
brew services start elasticsearch-full
# or
brew install opensearch
brew services start opensearch
```
{: file="Local Machine"}

จากนั้นเพิ่มบรรทัดเหล่านี้เข้าไปใน `Gemfile`:

```ruby
gem "searchkick"

gem "elasticsearch"   # select one
gem "opensearch-ruby" # select one
```
{: file="Gemfile"}

และเพิ่ม `searchkick` เข้าไปใน model ที่ต้องการจะค้นหา:

```ruby
class Product < ActiveRecord::Base
  searchkick
end
```
{: file="app/models/product.rb"}

macro `searchkick` ทำสามอย่าง:

- เพิ่ม class method `Product.search(query)` ที่ส่ง query ไปยัง search engine
- ผูก ActiveRecord callback ให้ auto-reindex ตอน save/destroy
- กำหนด default mapping ของ index (ปรับได้ด้วย option)

เข้าไปอ่าน [Searchkick documentation](https://github.com/ankane/searchkick) เพิ่มเติม ดูวิธีการใช้งานและ feature ที่มีให้ใช้

## Demo

ต่อไปเราจะไปลองทำ demo ง่าย ๆ กัน:

```sh
rails new store
```
{: file="Local Machine"}

จากนั้นก็สร้างโมเดล product ซึ่งขอลัด ๆ ใช้ scaffold ไปก่อน:

```sh
rails g scaffold Product name detail:text
rails db:migrate
rails s
```
{: file="Local Machine"}

สร้างตัวอย่างข้อมูลใน:

```ruby
Product.create(name: "", detail: "")
```
{: file="db/seeds.rb"}

จากนั้นก็รันคำสั่งเพื่อเพิ่มข้อมูลเข้าไป:

```sh
rails db:seed
```
{: file="Local Machine"}

เปิด `http://localhost:3000/products`:


![16988928511916](https://i.imgur.com/Z3dnbiv.png)


หลังจากนี้จะ implement search ให้กับ Product ก่อนอื่นก็เพิ่ม routes search:

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

เพิ่ม form การ search:

```erb
<%= form_tag search_products_path, method: :get do %>
  <%= text_field_tag :q %>
  <%= submit_tag :search %>
<% end %>
```
{: file="app/views/products/index.html.erb"}

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
{: file="app/controllers/products_controller.rb"}

`Product.search(params[:q])` คืน relation-like object ที่ chain method แบบ ActiveRecord ต่อได้ เช่น `.where(active: true)`, `.order(created_at: :desc)`, `.limit(10)` (ภายใน Searchkick แปลงเป็น Elasticsearch DSL ให้)

ตอนนี้ทำง่าย ๆ ไปก่อน แล้วไปลองดูผลงานกัน:


![16988930135463](https://i.imgur.com/tLidcrB.png)


มาถึงขั้นนี้แล้วก็เหมือนว่าทุกอย่างจะใช้ได้ดี ที่นี้ลองใส่ข้อมูลที่เป็นภาษาไทยลงไป:


![16988931025125](https://i.imgur.com/18Grhkl.png)


แล้วก็ลองค้นหาดู ก็จะเห็นว่าได้ผลออกมา:


![16988931625161](https://i.imgur.com/i7f48aY.png)


เกือบดีละ ที่นี้ลองค้นหาแค่บางส่วนของคำ:

![16988932201872](https://i.imgur.com/YS24nY5.png)


หาไม่เจอ 😓 มันเป็นปัญหาอย่างเดียวกับที่เคยเจอตอนใช้ [Thinking Sphinx](/posts/full-text-search-with-thinking-sphinx/) เป็นเรื่องการตัดคำภาษาไทยอีกแล้ว

## ตัดคำภาษาไทย (อัพเดท 2020)

> Elasticsearch/OpenSearch มี `thai` tokenizer built-in มาให้ (ใช้ ICU dictionary-based segmentation) ดังนั้นไม่ต้องเขียน tokenizer เองเหมือนตอนใช้ Thinking Sphinx ที่ต้องอาศัย gem [`thbrk`](https://github.com/phuwanart/thbrk) มาช่วย
{: .prompt-tip }

แก้ไข Searchkick ใน model โดยเพิ่ม mapping สำหรับภาษาไทยเข้าไป ต้องกำหนดว่าจะให้ค้นหาใน column ไหนได้บ้าง — อย่างในตัวอย่างคือให้ค้นหาที่ `name` และ `detail`:

```ruby
class Product < ApplicationRecord
  searchkick merge_mappings: true,
             settings: {
               analysis: {
                 analyzer: {
                   thai_analyzer: {
                     tokenizer: "thai"
                   }
                 }
               }
             },
             mappings: {
               properties: {
                 name: {
                   type: "keyword",
                   fields: {
                     analyzed: {
                       type: "text",
                       analyzer: "thai_analyzer"
                     }
                   }
                 },
                 detail: {
                   type: "keyword",
                   fields: {
                     analyzed: {
                       type: "text",
                       analyzer: "thai_analyzer"
                     }
                   }
                 }
               }
             }
end
```
{: file="app/models/product.rb"}

อธิบาย config:

- **`merge_mappings: true`** — รวม mapping ของเรากับ default ของ Searchkick (ไม่งั้น override ทั้งหมดและ feature default ของ Searchkick หาย)
- **`thai_analyzer` ใช้ `tokenizer: "thai"`** — analyzer คือชุดของ tokenizer + filter Elasticsearch มี `thai` tokenizer built-in ที่ตัดคำไทยให้อัตโนมัติ
- **`type: "keyword"` + `fields.analyzed.type: "text"`** — เก็บค่าเป็น 2 รูปแบบในไฟล์ index เดียว: ตัว keyword ไว้ exact match/sort/aggregation, ตัว text (`name.analyzed`) ไว้ search แบบ tokenize ภาษาไทย

จากนั้น reindex อีกครั้ง (จำเป็นทุกครั้งที่เปลี่ยน mapping):

```sh
rails searchkick:reindex CLASS=Product
```
{: file="Local Machine"}

ลองค้นหาผลลัพธ์ดู:

![16988932763830](https://i.imgur.com/q7vLWyb.png)

เป็นอันเรียบร้อย ทำให้ค้นหาแบบตัดคำภาษาไทยได้แล้ว 😉

## Conclusion

OpenSearch (หรือ Elasticsearch) เป็น search engine ที่ดี และ Searchkick ก็ทำให้มันใช้ง่าย ๆ กับ Rails model และตอนนี้ก็หาวิธีการให้ค้นหาภาษาไทยได้อย่างมีประสิทธิภาพได้แล้ว เพราะงั้นงานต่อไปได้ใช้อย่างแน่นอน

> **อัพเดท 2020:** Elasticsearch ไม่สามารถค้นหาภาษาไทยที่เป็นทับศัพท์ได้ (transliterated word) น่าจะเพราะการตัดคำของ tokenizer หากจะเลือกใช้ก็อย่าลืมดูเรื่องนี้ด้วย
{: .prompt-warning }

สำหรับการเปรียบเทียบกับ search engine ทางเลือกอื่น ดูที่โพสต์ [Full text search with Thinking Sphinx](/posts/full-text-search-with-thinking-sphinx/) ซึ่งมี section "ทางเลือกอื่นๆ" ที่เทียบ Searchkick/OpenSearch กับ pg_search, Meilisearch, Typesense ฯลฯ

## References

- [Searchkick README](https://github.com/ankane/searchkick)
- [Elasticsearch — Thai analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html#thai-analyzer)
- [OpenSearch — Text analysis](https://docs.opensearch.org/latest/analyzers/)
- [thbrk gem (Thai word breaker for Sphinx)](https://github.com/phuwanart/thbrk)
