---
layout: post
title: Ignore upgrade formula on brew
date: 2023-11-17 14:22 +0700
categories: Posts
tags: brew macos
---

ในบางครั้งเวลาอัพเกรด brew แล้วเจอบาง formula ที่อัพเกรดไม่ได้ — จะด้วยว่า macOS เก่าไป, version ใหม่มี breaking change, หรืออยากให้ version ตรงกับ team/CI ก็ตามแต่ เราสามารถ ignore มันได้โดยใช้คำสั่ง:

```sh
brew pin [formula]
```

ดูว่า pin อะไรไว้บ้าง:

```sh
brew list --pinned
```

และหากต้องการจะนำมาอัพเกรดอีกครั้งก็สั่ง:

```sh
brew unpin [formula]
```

> `brew pin` ป้องกันเฉพาะ `brew upgrade` และ `brew upgrade <formula>` เท่านั้น ถ้ารัน `brew install --upgrade <formula>` หรือ `brew reinstall` ตรงๆ มันยังอัพเกรดได้
{: .prompt-info }

## ทางเลือก

**Versioned formula** — ถ้าอยากใช้ version เฉพาะระยะยาว install ตรงไปจะคลีนกว่า pin:

```sh
brew install postgresql@14
brew install node@20
```

**`HOMEBREW_NO_AUTO_UPDATE=1`** — กัน brew auto-update ทุกครั้งที่รัน install (เร็วขึ้นด้วย) ใส่ใน `~/.zshrc` หรือ `~/.bashrc`:

```sh
export HOMEBREW_NO_AUTO_UPDATE=1
```
{: file="~/.zshrc"}

บางทีก็รำคาญเวลามัน log error ออกมาเยอะ

## References

- [Homebrew documentation — `brew pin`](https://docs.brew.sh/Manpage#pin-installed_formula-)
- [Homebrew environment variables](https://docs.brew.sh/Manpage#environment)
