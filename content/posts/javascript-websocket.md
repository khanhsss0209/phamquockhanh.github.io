---
title: "WebSocket cơ bản trong JavaScript"
date: 2025-10-23
draft: false
tags: ["JavaScript", "WebSocket", "Networking"]
---

**WebSocket** là giao thức giúp client và server **trao đổi dữ liệu hai chiều (real-time)**.  
Khác với HTTP, WebSocket không cần mở lại kết nối mỗi lần gửi dữ liệu.

Ví dụ:
```javascript
const socket = new WebSocket("wss://echo.websocket.org");

socket.onopen = () => socket.send("Xin chào Server!");
socket.onmessage = (e) => console.log("Server gửi:", e.data);