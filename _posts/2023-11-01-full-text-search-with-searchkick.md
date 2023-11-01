---
layout: post
title: Full text search with Searchkick
date: 2023-11-01 13:16 +0700
categories: Articles
tags: [opensearch, searchkick]
---

## Getting Started

Install [Elasticsearch](https://www.elastic.co/downloads/elasticsearch) or [OpenSearch](https://opensearch.org/downloads.html). For Homebrew, use:

```bash
brew install elastic/tap/elasticsearch-full
brew services start elasticsearch-full
# or
brew install opensearch
brew services start opensearch
```
Add these lines to your application’s Gemfile:

```ruby
gem "searchkick"

gem "elasticsearch"   # select one
gem "opensearch-ruby" # select one
```
{:file='Gemfile'}

```ruby
class Product < ActiveRecord::Base
  searchkick
end
```
{:file='app/models/product.rb'}

```bash
rails new store
```

```bash
rails g scaffold Product name detail:text
rails db:migrate
rails s
```

```ruby
Product.create(name: '', detail: '')
```
{:file='db/seeds.rb'}

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

```erb
<%= form_tag search_products_path, method: :get do %>
  <%= text_field_tag :q %>
  <%= submit_tag :search %>
<% end %>
```
{:file='app/views/products/index.html.erb'}

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
