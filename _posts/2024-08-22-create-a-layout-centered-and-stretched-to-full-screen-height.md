---
layout: post
title: Create a layout centered and stretched to full screen height
categories: Posts
tags: css html
date: 2024-08-22 13:46 +0700
---

รูปแบบหน้าจอที่ใช้บ่อยและแทบจะเป็นรูปแบบเริ่มต้นที่ผมใช้บ่อยจะเป็นลักษณะนี้:

```
------------
   header
------------

    main

------------
   footer
------------
```

ลองจินตนาการดูว่าในตอนแรกที่เราทำ ทุกอย่างจะไปอยู่ด้านบนหมด แน่นอนว่าหากมี content ยาวมากพอ ก็จะไม่ดูแปลกอะไร แต่หาก content สั้น footer จะลอยอยู่กลางหน้า ไม่ติดขอบล่าง

เพื่อให้ได้ในสิ่งที่ต้องการ เราจะกำหนด css พื้นฐานที่เราพยายามจะไม่อิงกับ css framework ตัวไหน ๆ เลย ก็จะได้ออกมาดังนี้:

```css
html {
  height: 100%;
}

body {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  min-height: 100dvh; /* handles mobile address bar automatically */
}

main {
  flex: 1 1;
}
```
{: file="app.css"}

หลักการสั้นๆ:

- **`display: flex` + `flex-direction: column`** บน `body` ทำให้ลูกแต่ละตัวเรียงต่อกันลงในแนวตั้ง
- **`min-height: 100dvh`** ตั้ง min height ของ body เท่าความสูง viewport ปัจจุบัน (`dvh` = dynamic viewport height ที่ปรับตาม mobile address bar ให้อัตโนมัติ) บรรทัด `100vh` เก็บไว้เป็น fallback สำหรับ browser เก่า
- **`flex: 1 1`** บน `main` บอกให้กินพื้นที่เหลือทั้งหมดที่ header กับ footer ไม่ได้ใช้ ดัน footer ไปติดขอบล่าง

> ในโพสเดิมเคยใช้ `min-height: -webkit-fill-available` เป็น workaround สำหรับ mobile viewport ตอนนี้ browser สมัยใหม่รองรับ `dvh`/`svh`/`lvh` หมดแล้ว ใช้ `100dvh` ตรงๆ คลีนกว่า
{: .prompt-tip }

รูปแบบ html ง่ายสุดที่เราจะใช้คือ:

```html
<header>
  <nav></nav>
</header>
<main></main>
<footer></footer>
```
{: file="index.html"}

ใช้ `<main>` แทน `<div class="main">` เพราะเป็น semantic HTML5 — screen reader จับ landmark ได้ ไม่ต้องประกาศ class เพิ่ม

เมื่อเราดูผลลัพธ์ที่ออกมาจะได้ว่าส่วน main จะยืดและดันให้ header และ footer ติดขอบบนและล่าง ซึ่งในระหว่าง main กับ header และ footer เราก็สามารถที่จะแทรก section หรืออะไรที่เราอยากจะแทรกไปได้เลย:

```html
<header>
  <nav></nav>
</header>
<div></div>
<main></main>
<section></section>
<footer></footer>
```
{: file="index.html"}

การแสดงผลก็ยังจะเต็มหน้าจอยู่เหมือนเดิม จนว่าจะมี content มากจน main ยืดอีกไม่ได้แล้ว จากนั้นหน้าก็จะเลื่อนลงได้

## ทางเลือก: CSS Grid

ถ้าชอบ grid syntax อ่านง่ายกว่าก็ใช้แทนได้:

```css
body {
  display: grid;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
  min-height: 100dvh;
}
```
{: file="app.css"}

- **`auto`** แถวแรกและแถวสุดท้าย — สูงเท่ากับ content (header กับ footer)
- **`1fr`** ตรงกลาง — กินพื้นที่เหลือทั้งหมด

HTML ใช้แบบเดียวกัน ไม่ต้องเปลี่ยน

## References

- [MDN — Basic concepts of flexbox](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_flexible_box_layout/Basic_concepts_of_flexbox)
- [MDN — Viewport units (`dvh`, `svh`, `lvh`)](https://developer.mozilla.org/en-US/docs/Web/CSS/length#viewport-percentage_lengths)
- [MDN — `<main>` element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/main)
