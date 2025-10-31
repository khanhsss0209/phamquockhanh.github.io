---
title: "Fetch API và AJAX trong JavaScript"
date: 2025-10-23
draft: false
tags: ["JavaScript", "HTTP", "Web"]
---

Trước đây, AJAX được dùng để gửi yêu cầu HTTP từ trình duyệt mà không tải lại trang.  
Ngày nay, ta dùng **Fetch API** — cú pháp gọn và hiện đại hơn.

Ví dụ:
```javascript
fetch('https://jsonplaceholder.typicode.com/posts/1')
  .then(res => res.json())
  .then(data => console.log(data));

Hoặc với async/await:
  const data = await fetch('https://jsonplaceholder.typicode.com/posts')
  .then(r => r.json());
console.log(data);
