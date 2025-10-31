---
title: "Kết nối Socket trong Java"
date: 2025-10-23
draft: false
tags: ["Java", "Socket"]
---

Trong Java, **Socket** được sử dụng để tạo kết nối giữa client và server qua giao thức TCP.  
Ví dụ đơn giản:

```java
// Client.java
import java.io.*;
import java.net.*;

public class Client {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("localhost", 1234);
        PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
        out.println("Xin chào Server!");
        socket.close();
    }
}

// Server.java
import java.io.*;
import java.net.*;

public class Server {
    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket(1234);
        Socket client = server.accept();
        BufferedReader in = new BufferedReader(new InputStreamReader(client.getInputStream()));
        System.out.println("Client gửi: " + in.readLine());
        server.close();
    }
}
Chạy Server trước, rồi chạy Client — bạn sẽ thấy dữ liệu được gửi qua socket.