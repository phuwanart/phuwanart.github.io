---
layout: post
title: Install Flowbite on Rails 8
categories:
- Notes
tags:
- rails
- rails 8
- flowbite
date: 2025-04-14 19:20 +0700
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

```css
@import "../../../node_modules/flowbite/src/themes/default";

@plugin "../../../node_modules/flowbite/plugin";

@source "../../../node_modules/flowbite";
```
{:file='app/assets/tailwind/application.css'}

```js
import "../../node_modules/flowbite/dist/flowbite"
```
{:file='app/javascript/application.js'}

