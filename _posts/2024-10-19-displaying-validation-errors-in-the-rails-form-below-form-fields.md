---
layout: post
title: Displaying validation errors in the rails form below form fields
categories: Posts
tags: rails
date: 2024-10-19 12:45 +0700
---
หากใช้ scaffold จะได้ form ตั้งต้นในการใส่ข้อมูล ซึ่งหากมี valicate จะมีการแสดง error ตามรูป

![](https://i.imgur.com/YnOXcgI.png)

ซึ่งหากเราต้องการให้ข้อความ error ไปอยู่ในแต่ละ filed นั้น สามารถทำได้ตามข้างล่าง:

```diff
  <%= form.label :title %>
  <%= form.text_field :title, class: "block shadow rounded-md border border-gray-400 outline-none px-3 py-2 mt-2 w-full" %>
+ <% post.errors.full_messages_for(:title).each do |message| %>
+   <%= message %>
+ <% end %>
```
{:file='app/views/posts/_form.html.erb'}

แค่นี้ error ก็จะแสดงในส่วนของ field นั้น ๆ แล้ว

![](https://i.imgur.com/f1B3rCD.png)

เป็นโพสสั้น ๆ แต่ขอเขียนหน่อย เพราะมักจะลืมว่าทำยังไง 😅