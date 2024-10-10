---
layout: post
title: Displaying validation errors in the rails form below form fields
categories: Posts
tags: rails
---

```sh
rails new demo -c tailwind
```

```sh
rails g scaffold Post title body:text
```

```ruby
class Post < ApplicationRecord
  validates :title, presence: true
end
```

![](https://i.imgur.com/YnOXcgI.png)

```diff
  <%= form.label :title %>
  <%= form.text_field :title, class: "block shadow rounded-md border border-gray-400 outline-none px-3 py-2 mt-2 w-full" %>
+ <% post.errors.full_messages_for(:title).each do |message| %>
+   <%= message %>
+ <% end %>
```
{:file='app/views/posts/_form.html.erb'}

![](https://i.imgur.com/f1B3rCD.png)