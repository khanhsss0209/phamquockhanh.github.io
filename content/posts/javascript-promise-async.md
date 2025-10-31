---
title: "Promise và Async/Await trong JavaScript"
date: 2025-10-23
draft: false
tags: ["JavaScript", "Async", "Promise"]
---

`Promise` giúp xử lý các tác vụ bất đồng bộ gọn gàng hơn `callback`.

Ví dụ Promise cơ bản:
```javascript
let p = new Promise((resolve) => {
  setTimeout(() => resolve("Hoàn thành!"), 1000);
});
p.then(result => console.log(result));

Còn async/await giúp viết code bất đồng bộ trông giống đồng bộ:

async function run() {
  let result = await p;
  console.log(result);
}
run();