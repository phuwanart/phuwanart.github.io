---
layout: post
title: Install Flowbite on Rails 8
categories:
- Posts
tags:
- rails
- rails 8
- flowbite
- tailwindcss
date: 2025-04-14 18:59 +0700
---

[Flowbite](https://flowbite.com/) เป็นชุด component บน Tailwind CSS ที่มีทั้งแบบ free และ pro มีจำนวน component และ template ค่อนข้างเยอะ ทำงานด้วย vanilla JavaScript จึงเข้ากันได้ดีกับ Rails ที่ใช้ Hotwire โพสต์นี้พาติดตั้ง Flowbite บน Rails 8 ใหม่ที่ใช้ Tailwind v4 และ Bun พร้อมแก้จุดที่ component หยุดทำงานหลัง Turbo navigation

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
- **`-j bun`** — ใช้ jsbundling-rails กับ Bun แทน esbuild หรือ rollup

## ย้าย Tailwind entrypoint

ค่า default ของ Rails จะวาง `application.tailwind.css` ไว้ใน `app/assets/stylesheets/` ซึ่งเป็น folder เดียวกับ asset อื่นๆ ของแอป ย้ายไปแยกใน `app/assets/tailwind/` เพื่อให้ source CSS ของ Tailwind ไม่ปนกับ stylesheet ของแอป:

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

## ติดตั้ง Flowbite

```sh
bun add flowbite
```

ประกาศให้ Tailwind รู้จัก Flowbite ใน entrypoint CSS:

```css
@import "../../../node_modules/flowbite/src/themes/default";

@plugin "../../../node_modules/flowbite/plugin";

@source "../../../node_modules/flowbite";
```
{: file="app/assets/tailwind/application.css"}

อธิบายแต่ละ directive:

- **`@import ".../flowbite/src/themes/default"`** — โหลด design token เริ่มต้น (สี, spacing, typography) ของ Flowbite เข้ามา ถ้าอยาก customize theme ก็ override ตัวแปร CSS ที่อยู่ในนี้
- **`@plugin ".../flowbite/plugin"`** — register Flowbite เป็น Tailwind plugin (syntax ของ Tailwind v4) เพื่อให้ utility class ของ Flowbite ถูก generate ออกมาตอน build
- **`@source ".../flowbite"`** — บอก Tailwind v4 ให้ scan class ในไฟล์ของ Flowbite ด้วย ไม่งั้น class ที่ปรากฏแค่ใน JS หรือ template ของ Flowbite จะถูก purge ออก

## Import Flowbite JS และผูกกับ Turbo

Import Flowbite ใน JavaScript entrypoint:

```js
import "../../node_modules/flowbite/dist/flowbite"
```
{: file="app/javascript/application.js"}

> Flowbite init component ครั้งเดียวตอน page load ถ้าใช้ Turbo แล้ว navigate ข้ามหน้า DOM ใหม่จะไม่ได้รับการ init ทำให้ dropdown, modal, tooltip ฯลฯ หยุดทำงาน ต้องเรียก `initFlowbite()` ใหม่ทุกครั้งที่ Turbo render
{: .prompt-warning }

เพิ่ม event listener สำหรับ Turbo:

```js
import { initFlowbite } from "flowbite"

document.addEventListener("turbo:load", () => {
  initFlowbite()
})

document.addEventListener("turbo:render", () => {
  initFlowbite()
})
```
{: file="app/javascript/application.js"}

`turbo:load` ครอบ navigation ปกติ ส่วน `turbo:render` ครอบกรณี Turbo Frame หรือ Turbo Stream ที่ update DOM แค่บางส่วน

## ทดสอบ

สร้าง home controller สำหรับลอง component:

```sh
rails g controller home index
```

Copy markup component อย่าง [Dropdown](https://flowbite.com/docs/components/dropdowns/) หรือ [Modal](https://flowbite.com/docs/components/modal/) จาก Flowbite docs มาวางใน `app/views/home/index.html.erb` แล้วชี้ root route ไปที่ `home#index`

รัน Rails server พร้อม watcher ของ Tailwind และ Bun:

```sh
bin/dev
```

เปิด `http://localhost:3000` ทดสอบ component ที่มี state เช่น dropdown และ navigate ระหว่างหน้าผ่าน link เพื่อยืนยันว่า Flowbite ยังทำงานหลัง Turbo navigation

## Conclusion

Rails 8 app พร้อมใช้ Flowbite component ได้แล้ว สิ่งที่ต้องระวังคือ `initFlowbite()` ต้องถูกเรียกใหม่ทุกครั้งหลัง Turbo render ไม่งั้น component ที่ต้องใช้ JS จะหยุดทำงานหลัง navigate

จะเลือก Flowbite หรือ [Preline](/posts/install-preline-ui-on-rails-8/) ขึ้นอยู่กับ:

- **License + budget** — Flowbite มี tier pro ที่ต้องจ่ายเงินสำหรับ component และ template ขั้นสูง ส่วน Preline เป็น open-source ทั้งหมด
- **Design language** — ลองเทียบ visual style ของทั้งสองแล้วเลือกที่เข้ากับโปรเจกต์มากกว่า
- **Ecosystem** — Flowbite เก่ากว่า มี community และ template สำเร็จเยอะกว่า ส่วน Preline ออกแบบให้รัน autoInit ง่ายและ component crisp กว่า

## References

- [Flowbite documentation](https://flowbite.com/docs/getting-started/introduction/)
- [Flowbite + Tailwind v4 install guide](https://flowbite.com/docs/getting-started/tailwind-css/)
- [Tailwind CSS v4 upgrade guide](https://tailwindcss.com/docs/upgrade-guide)
- [Rails 8 cssbundling-rails](https://github.com/rails/cssbundling-rails)
- [jsbundling-rails with Bun](https://github.com/rails/jsbundling-rails)
