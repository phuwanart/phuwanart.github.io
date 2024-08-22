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

ลองจิตนาการดูว่าในตอนแรกที่เราทำ ทุกอย่างจะไปอยู่ด้านบนหมด แน่นอนว่าหากมี content ยาวมากพอ ก็จะไม่ดูแปลกอะไร

เพื่อให้ได้ในสิ่งที่ต้องการ เราจะกำหนด css พื้นฐานที่เราพยายามจะไม่อิงกับ css framework ตัวไหน ๆ เลย ก็จะได้ออกมาดังนี้:

```css
html {
  height: 100%;
}

body {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  /* mobile viewport bug fix */
  min-height: -webkit-fill-available;
}

.main {
  flex: 1 1;
}
```

และรูปแบบ html ง่ายสุดที่เราจะใช้คือ:

```html
<header>
  <nav></nav>
</header>
<div class="main"></div>
<footer></footer>
```

เมื่อเราดูผลลัพท์ที่ออกมาจะได้ว่าส่วน main จะยืดและดันให้ header และ footer ให้ติดขอบบนและล่าง ซึ่งในระหว่าง main กับ header และ footer เราก็สามารถที่จะแทรก section หรืออะไรที่เราอยากจะแทรกไปได้เลย 

```html
<header>
  <nav></nav>
</header>
<div></div>
<div class="main"></div>
<section></section>
<footer></footer>
```

การแสดงผลก็ยังจะเต็มหน้าจอยู่เหมือนเดิม จนว่าจะมี content มากจน main ยืดอีกไม่ได้แล้ว จากนั้นหน้าก็จะเลื่อนลงได้