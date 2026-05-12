---
layout: post
title: Install Preline UI on Rails 8
categories:
- Posts
tags:
- rails
- rails 8
- tailwindcss
- preline
date: 2025-04-14 18:59 +0700
---

[Preline UI](https://preline.co/) เป็นชุด component แบบ open-source ที่สร้างบน Tailwind CSS โดยไม่ต้องพึ่ง JavaScript framework ใดๆ ทำให้เข้ากันได้ดีกับ Rails ที่ใช้ Hotwire (Turbo + Stimulus) โพสต์นี้พาติดตั้ง Preline บน Rails 8 ใหม่ที่ใช้ Tailwind v4 และ Bun เป็น JavaScript runtime พร้อมแก้จุดที่ Preline component หยุดทำงานหลัง Turbo navigation

## Prerequisites

- Ruby 3.3+
- Rails 8
- ติดตั้ง [Bun](https://bun.sh/) บนเครื่อง dev
- ความคุ้นเคยกับ Tailwind CSS เบื้องต้น

## สร้าง Rails app

```sh
rails new blog -c tailwind -j bun
```

- **`-c tailwind`** — ใช้ cssbundling-rails กับ Tailwind (ได้ Tailwind v4 มาเลย)
- **`-j bun`** — ใช้ jsbundling-rails กับ Bun แทน esbuild หรือ rollup (ติดตั้งและ build เร็วกว่าเดิมพอสมควร)

## ย้าย Tailwind entrypoint

ค่า default ของ Rails จะวาง `application.tailwind.css` ไว้ใน `app/assets/stylesheets/` ซึ่งเป็น folder เดียวกับ asset อื่นๆ ของแอป ย้ายไปแยกใน `app/assets/tailwind/` เพื่อให้ source CSS ของ Tailwind ไม่ปนกับ stylesheet ของแอป (และไม่โดน asset pipeline หยิบไปใช้โดยไม่ตั้งใจ):

```sh
mkdir app/assets/tailwind
mv app/assets/stylesheets/application.tailwind.css app/assets/tailwind/application.css
```

อัปเดต path ใน `build:css` script ของ `package.json` ให้ตรงกับ entrypoint ใหม่:

```diff
 {
   "name": "app",
   "private": true,
   "scripts": {
     "build": "bun bun.config.js",
-    "build:css": "bunx @tailwindcss/cli -i ./app/assets/stylesheets/application.tailwind.css -o ./app/assets/builds/application.css --minify"
+    "build:css": "bunx @tailwindcss/cli -i ./app/assets/tailwind/application.css -o ./app/assets/builds/application.css --minify"
   },
   "dependencies": {
     "@hotwired/stimulus": "^3.2.2",
     "@hotwired/turbo-rails": "^8.0.13",
     "@tailwindcss/cli": "^4.1.3",
     "tailwindcss": "^4.1.3"
   }
 }
```
{: file="package.json"}

## ติดตั้ง Preline

```sh
bun add preline
```

ประกาศให้ Tailwind รู้จัก Preline ใน entrypoint CSS:

```css
/* Preline UI */
@import "../../../node_modules/preline/variants.css";
@source "../../../node_modules/preline/dist/*.js";

/* Optional plugins */
/* @import "../../../node_modules/preline/src/plugins/overlay/variants.css"; */
```
{: file="app/assets/tailwind/application.css"}

- **`@import "preline/variants.css"`** — โหลด custom variant ของ Preline (เช่น state ต่างๆ ของ dropdown, modal) เข้ามาใช้ใน Tailwind
- **`@source ".../preline/dist/*.js"`** — บอก Tailwind v4 ให้ scan class ในไฟล์ JS ของ Preline ด้วย ไม่งั้น class ที่ปรากฏแค่ใน JS จะถูก purge ออกตอน build

## ติดตั้ง Tailwind plugin เสริม

Preline component หลายตัวพึ่ง form styling และ aspect ratio ติดตั้ง plugin ที่ Tailwind จัดให้:

```sh
bun add @tailwindcss/forms @tailwindcss/aspect-ratio
```

เปิด plugin ใน CSS entrypoint:

```css
/* Third-party plugins */
@plugin "@tailwindcss/forms";
@plugin "@tailwindcss/aspect-ratio";
```
{: file="app/assets/tailwind/application.css"}

## ปรับ default behavior

Tailwind v4 ปิด hover style บน device ที่ไม่มี hover (mobile/tablet) ตาม spec ใหม่ ซึ่งหลายครั้งทำให้ UI บนมือถือดูแห้งไป ถ้าอยากให้ hover ทำงานทุกอุปกรณ์ (เหมือนพฤติกรรม Tailwind v3) override custom variant และเพิ่ม cursor ให้ button ไปด้วย:

```css
/* Adds pointer cursor to buttons */
@layer base {
  button:not(:disabled),
  [role="button"]:not(:disabled) {
    cursor: pointer;
  }
}

/* Defaults hover styles on all devices */
@custom-variant hover (&:hover);
```
{: file="app/assets/tailwind/application.css"}

## Import Preline JS และผูกกับ Turbo

Import Preline ใน JavaScript entrypoint:

```js
import "../../node_modules/preline/dist/preline"
```
{: file="app/javascript/application.js"}

> Preline init component ครั้งเดียวตอน page load ถ้าใช้ Turbo แล้ว navigate ข้ามหน้า DOM ใหม่จะไม่ได้รับการ init ทำให้ dropdown, modal, accordion ฯลฯ หยุดทำงาน ต้องเรียก `HSStaticMethods.autoInit()` ใหม่ทุกครั้งที่ Turbo render
{: .prompt-warning }

เพิ่ม event listener สำหรับ Turbo:

```js
import "../../node_modules/preline/dist/preline"

document.addEventListener("turbo:load", () => {
  window.HSStaticMethods.autoInit()
})

document.addEventListener("turbo:render", () => {
  window.HSStaticMethods.autoInit()
})
```
{: file="app/javascript/application.js"}

`turbo:load` ครอบ navigation ปกติ ส่วน `turbo:render` ครอบกรณี Turbo Frame หรือ Turbo Stream ที่ update DOM แค่บางส่วน

## ทดสอบ

สร้าง home controller สำหรับลอง component:

```sh
rails g controller home index
```

Copy markup จาก [Application Layouts](https://preline.co/examples/layouts-application.html) ของ Preline มาวางใน `app/views/home/index.html.erb` แล้วชี้ root route ไปที่ `home#index`

รัน Rails server พร้อม watcher ของ Tailwind และ Bun:

```sh
bin/dev
```

เปิด `http://localhost:3000` ทดสอบ component ที่มี state เช่น dropdown หรือ modal และลอง navigate ระหว่างหน้าผ่าน link เพื่อยืนยันว่า Preline ยังทำงานหลัง Turbo navigation

## Conclusion

ตอนนี้ Rails 8 app พร้อมใช้ Preline component ทั้งชุดได้แล้ว จุดที่ต้องระวังหลักๆ มีสองอย่าง — Tailwind v4 มี breaking change หลายจุด (default ของ hover, syntax ของ `@plugin` / `@custom-variant`) และ Preline ต้องการ re-init ทุกครั้งหลัง Turbo navigation ถ้าจำสองข้อนี้ได้ component ที่เหลือใช้ตรงๆ ตาม example บน Preline website

## References

- [Preline UI documentation](https://preline.co/docs/index.html)
- [Preline + Tailwind v4 install guide](https://preline.co/docs/v2/frameworks-tailwindcss-v4.html)
- [Tailwind CSS v4 upgrade guide](https://tailwindcss.com/docs/upgrade-guide)
- [Rails 8 cssbundling-rails](https://github.com/rails/cssbundling-rails)
- [jsbundling-rails with Bun](https://github.com/rails/jsbundling-rails)
