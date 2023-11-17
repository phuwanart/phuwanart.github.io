---
layout: post
title: Ignore upgrade formula on brew
date: 2023-11-17 14:22 +0700
categories: Posts
tags: brew
---

ในบางครั้งเวลาอัพเกรด brew แล้วเจอบาง formula ที่อัพเกรดไม่ได้ จะด้วยว่าเพราะ os เก่าไปหรือว่าอะไรก็ตามแต่ เราสามารถ ignore มันได้โดยใช้คำสั่ง:

```bash
brew pin [formula]
```

และหากต้องการจะนำมาอัพเกรดอีกครั้งก็สั่ง:

```bash
brew unpin [formula]
```

บางทีก็รำคาญเวลามัน log error ออกมาก
