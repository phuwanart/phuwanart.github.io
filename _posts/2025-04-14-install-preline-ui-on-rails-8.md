---
layout: post
title: Install Preline UI on Rails 8
categories:
- Notes
tags:
- rails
- rails 8
- tailwindcss
- preline
date: 2025-04-14 18:59 +0700
---

```sh
rails new blog -c tailwind -j bun
```

```sh
mkdir app/assets/tailwind
mv app/assets/stylesheets/application.tailwind.css app/assets/tailwind/application.css
```

```diff
{
  "name": "app",
  "private": true,
  "scripts": {
    "build": "bun bun.config.js",
--  "build:css": "bunx @tailwindcss/cli -i ./app/assets/stylesheets/application.tailwind.css -o ./app/assets/builds/application.css --minify"
++. "build:css": "bunx @tailwindcss/cli -i ./app/assets/tailwind/application.css -o ./app/assets/builds/application.css --minify"
  },
  "dependencies": {
    "@hotwired/stimulus": "^3.2.2",
    "@hotwired/turbo-rails": "^8.0.13",
    "@tailwindcss/cli": "^4.1.3",
    "tailwindcss": "^4.1.3"
  }
}
```
{:file='package.json'}


```sh
bun add preline
```

```css
/* Preline UI */
@import "../../../node_modules/preline/variants.css";
@source "../../../node_modules/preline/dist/*.js";

/* Optional plugins */
/* @import "../../../node_modules/preline/src/plugins/overlay/variants.css"; */
```
{:file='app/assets/tailwind/application.css'}

```sh
bun add @tailwindcss/forms @tailwindcss/aspect-ratio
```

```css
/* Third-party plugins */
@plugin "@tailwindcss/forms";
@plugin "@tailwindcss/aspect-ratio";
```
{:file='app/assets/tailwind/application.css'}

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
{:file='app/assets/tailwind/application.css'}

```js
import "../../node_modules/preline/dist/preline"
```
{:file='app/javascript/application.js'}

```sh
rails g controller home index
```

copy html from [Application Layouts](https://preline.co/examples/layouts-application.html)
