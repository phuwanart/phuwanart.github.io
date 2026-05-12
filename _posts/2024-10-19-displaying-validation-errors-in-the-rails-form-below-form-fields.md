---
layout: post
title: Displaying validation errors in the rails form below form fields
categories: Posts
tags: rails
date: 2024-10-19 12:45 +0700
---

หากใช้ scaffold จะได้ form ตั้งต้นในการใส่ข้อมูล ซึ่งหากมี validate จะมีการแสดง error ตามรูป

![](https://i.imgur.com/YnOXcgI.png)

ซึ่งหากเราต้องการให้ข้อความ error ไปอยู่ในแต่ละ field นั้น สามารถทำได้ตามข้างล่าง:

```diff
   <%= form.label :title %>
   <%= form.text_field :title, class: "block shadow rounded-md border border-gray-400 outline-none px-3 py-2 mt-2 w-full" %>
+  <% post.errors.full_messages_for(:title).each do |message| %>
+    <%= message %>
+  <% end %>
```
{: file="app/views/posts/_form.html.erb"}

`errors.full_messages_for(:title)` คืน array ของข้อความเต็มที่ผูกกับ attribute นั้น เช่น `["Title can't be blank"]` (ส่วน `errors[:title]` จะคืนแค่ `["can't be blank"]` โดยไม่มีชื่อ attribute นำ)

แค่นี้ error ก็จะแสดงในส่วนของ field นั้น ๆ แล้ว

![](https://i.imgur.com/f1B3rCD.png)

## เพิ่ม styling ให้เห็นเป็น error ชัดๆ

ห่อ message ด้วย tag ที่สีและขนาดบอกว่าเป็น error:

```erb
<% post.errors.full_messages_for(:title).each do |message| %>
  <p class="text-red-500 text-sm mt-1"><%= message %></p>
<% end %>
```
{: file="app/views/posts/_form.html.erb"}

## เพิ่ม accessibility

ผูก field กับ error message ผ่าน `aria-describedby` ให้ screen reader อ่าน error ตอน focus เข้า field และตั้ง `aria-invalid` ตอนมี error:

```erb
<%= form.text_field :title,
      class: "block shadow rounded-md border border-gray-400 outline-none px-3 py-2 mt-2 w-full",
      "aria-invalid": post.errors[:title].any?,
      "aria-describedby": "post_title_errors" %>

<div id="post_title_errors">
  <% post.errors.full_messages_for(:title).each do |message| %>
    <p class="text-red-500 text-sm mt-1"><%= message %></p>
  <% end %>
</div>
```
{: file="app/views/posts/_form.html.erb"}

เป็นโพสสั้น ๆ แต่ขอเขียนหน่อย เพราะมักจะลืมว่าทำยังไง 😅

## References

- [Active Record Validations — Displaying Validation Errors in Views](https://guides.rubyonrails.org/active_record_validations.html#displaying-validation-errors-in-views)
- [ARIA: aria-describedby](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-describedby)
